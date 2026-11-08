---
title: "S3 TablesのHiveテーブル→Icebergマネージド移行を試す（Sync・移行ポリシーの挙動まで）"
emoji: "🧊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "s3", "s3tables", "iceberg", "athena"]
published: false
---
<!--
執筆メモ（公開前に削除）
- 「（検証後に記載）」の箇所に結果・スクリーンショット・所要時間を埋める
- 想定と違う挙動が出たら、該当の「確認したいこと」と並べて書く
- 画像は /images/s3tables-hive-to-iceberg-migration/ に置く
-->

# はじめに
2026年11月に、Amazon S3 Tables で AWS Glue Data Catalog に登録された Apache Hive テーブルを Apache Iceberg テーブルへ移行する機能（マネージド移行）が提供されました。

https://aws.amazon.com/jp/blogs/news/amazon-s3-tables-now-offer-managed-migration-of-apache-hive-tables-to-apache-iceberg/

これまで Hive テーブルを Iceberg に移行するには、Athena の CTAS や Spark のジョブを自前で用意し、型の変換や小さいファイルの統合、移行中に増えたデータの追従まで考える必要がありました。
今回の機能では、S3 Tables の API を呼ぶだけで、データのコピー・型変換・コンパクションが行われ、切り替えまでの差分は Sync で追従できます。

本記事では、基本的な Start → Sync → Complete の流れに加えて、ドキュメントに書かれている次の挙動を実際に確認してみます。

- 小さいファイルが移行先でどの程度まとめられるか
- Sync でソース側のファイルの追加・削除・上書きがどう扱われるか
- 移行ポリシー（`fail` / `drop` / `setNull`）で、余分な列や型の合わないファイルがどう扱われるか
- Complete の前後で、移行先テーブルへの書き込み可否がどう変わるか

<br>

# アップデート概要
ドキュメントから、検証に関係する仕様を整理しておきます。

| 項目 | 内容 |
| --- | --- |
| 移行元 | Glue Data Catalog に登録された Hive 外部テーブル（データは汎用バケット上の Parquet） |
| 移行先 | S3 Tables のテーブルバケット上の Iceberg テーブル（移行時に新規作成） |
| 制約 | 移行元・テーブルバケット・レポート用バケットは同一アカウント・同一リージョン |
| 対応リージョン | 東京を含む10リージョン |
| 操作 | コンソール / AWS CLI / S3 Tables API / Agent Toolkit のスキル |
| 料金 | 初回の移行（Start）は無料。Sync は移行ごとに最初の500回（開始から3か月以内）が無料で、以降は移行元オブジェクト数に応じて課金。移行先のストレージ・リクエストは通常の S3 Tables 料金 |

移行は次の3段階で進みます。

1. Start（`StartTableMigration`）：移行先テーブルを作成し、移行元のデータをコピーします。
2. Sync（`SyncTableMigration`）：前回の Start / Sync 以降に変わったオブジェクトだけをコピーします。何回でも実行できます。
3. Complete（`CompleteTableMigration`）：移行先テーブルを読み書き可能にし、テーブルメンテナンスを有効にします。取り消しはできません。

移行元のデータはコピーされるだけで、移行元テーブルはそのまま残ります。移行中も移行元への読み書きは続けられ、移行先は Complete まで読み取り専用になります。

Start と Sync の内部では、`planning`（移行元のオブジェクトを洗い出し、互換性レポートを出力）と `migrating`（データを変換・圧縮してコミット）の2つの状態を経ます。`reviewMigrationPlan` を `true` にすると、`planning` の後に `pendingReview` で止まり、レポートを確認してから承認できます。

また、移行対象外になる主な条件は次のとおりです。

- ORC・Avro・JSON などの Parquet 以外の形式、LZO 圧縮
- ACID（トランザクション）テーブル、すでに Iceberg などのテーブル形式になっているテーブル
- Lake Formation にも登録されている Glue テーブル
- Athena のパーティション射影を使うテーブル（パーティションを Glue に登録すれば対象になる）
- テーブルのロケーション外に置かれたパーティション
- SSE-C で暗号化されたオブジェクト、Glacier 系などの復元が必要なオブジェクト

Glue のテーブルパラメータ、SerDe 設定、列の統計情報、列コメント、テーブルの説明はコピーされません。

<br>

# 検証の構成
東京リージョン（ap-northeast-1）で、次の構成を作ります。

（検証後に構成図を記載）

| リソース | 名前（例） | 用途 |
| --- | --- | --- |
| 汎用バケット | `hive-mig-src-<アカウントID>` | 移行元の Parquet ファイル |
| 汎用バケット | `hive-mig-report-<アカウントID>` | 移行レポートの出力先 |
| Glue データベース | `hive_mig_db` | 移行元の Hive テーブル |
| Glue テーブル | `orders` | 検証1・2・4用（きれいなデータ） |
| Glue テーブル | `orders_dirty` | 検証3用（余分な列・型の合わないファイルを含む） |
| テーブルバケット | `hive-mig-dest` | 移行先 |
| 名前空間 | `analytics` | 移行先テーブルを作る名前空間 |
| IAM ロール | `S3TablesMigrationRole` | 移行ジョブが引き受けるロール |

<br>

# 前提条件
以下は事前に済ませておく前提とします。

- AWS CLI v2 のインストール（`start-table-migration` が使えるバージョン。古いと `Found invalid choice` になるため、`aws s3tables start-table-migration help` が通ることを確認する）
- Python 3 と `pyarrow` のインストール（テストデータの生成に使用）
- Athena のクエリ結果の出力先の設定
- 作業ユーザーに、S3・Glue・S3 Tables・IAM・Athena を操作する権限があること
- テーブルバケットで AWS 分析サービスとの統合を有効にすること（Athena から移行先テーブルをクエリするため）

アカウントで Lake Formation の権限管理を有効にしている場合、Athena から移行先テーブルを参照するには、作業ユーザーへ Lake Formation の権限付与が必要になります。

<br>

# 検証環境の準備
## 変数の設定
以降のコマンドで使う値をシェル変数にしておきます。

```bash
export AWS_REGION=ap-northeast-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

export SRC_BUCKET=hive-mig-src-${ACCOUNT_ID}
export REPORT_BUCKET=hive-mig-report-${ACCOUNT_ID}
export GLUE_DB=hive_mig_db
export TABLE_BUCKET=hive-mig-dest
export NAMESPACE=analytics
export ROLE_NAME=S3TablesMigrationRole

aws --version
```

<br>

## 汎用バケットの作成
移行元とレポート用のバケットを作成します。

```bash
aws s3 mb s3://${SRC_BUCKET} --region ${AWS_REGION}
aws s3 mb s3://${REPORT_BUCKET} --region ${AWS_REGION}
```

<br>

## テストデータの生成
コンパクションの効果を見るため、1ファイルあたりの行数を少なくして、小さい Parquet ファイルを大量に作ります。
`orders` には3日分のパーティションに20ファイルずつ、計60ファイルを置きます。

`orders_dirty` には、正常なファイルに加えて、移行ポリシーの挙動を見るための2種類のファイルを置きます。

- `extra_column.parquet`：Glue のスキーマにない `audit` 列を持つファイル
- `type_mismatch.parquet`：Glue では `int` の `customer_id` を `bigint`（int64）で持つファイル

```python:gen_data.py
import os
import random
from datetime import datetime, timedelta

import pyarrow as pa
import pyarrow.parquet as pq

random.seed(0)
STATUSES = ["created", "paid", "shipped", "cancelled"]

def make_rows(start_id, n, dt):
    base = datetime.strptime(dt, "%Y-%m-%d")
    return {
        "order_id": list(range(start_id, start_id + n)),
        "customer_id": [random.randint(1, 10000) for _ in range(n)],
        "amount": [round(random.uniform(100, 50000), 2) for _ in range(n)],
        "status": [random.choice(STATUSES) for _ in range(n)],
        "ordered_at": [base + timedelta(seconds=random.randint(0, 86399)) for _ in range(n)],
    }

SCHEMA = pa.schema([
    ("order_id", pa.int64()),
    ("customer_id", pa.int32()),
    ("amount", pa.float64()),
    ("status", pa.string()),
    ("ordered_at", pa.timestamp("us")),
])

def write(path, rows, schema=SCHEMA):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    pq.write_table(pa.table(rows, schema=schema), path, compression="snappy")

if __name__ == "__main__":
    # orders: 3パーティション x 20ファイル x 500行（order_id 1〜30000）
    order_id = 1
    for dt in ["2026-11-01", "2026-11-02", "2026-11-03"]:
        for i in range(20):
            write(f"out/orders/dt={dt}/part-{i:03d}.parquet", make_rows(order_id, 500, dt))
            order_id += 500

    # orders_dirty: 正常ファイル x 5 + 異常ファイル x 2（order_id 30001〜33500）
    dt = "2026-11-01"
    for i in range(5):
        write(f"out/orders_dirty/dt={dt}/part-{i:03d}.parquet", make_rows(order_id, 500, dt))
        order_id += 500

    rows = make_rows(order_id, 500, dt)
    rows["audit"] = ["batch-x"] * 500
    write(f"out/orders_dirty/dt={dt}/extra_column.parquet", rows,
          SCHEMA.append(pa.field("audit", pa.string())))
    order_id += 500

    write(f"out/orders_dirty/dt={dt}/type_mismatch.parquet", make_rows(order_id, 500, dt),
          SCHEMA.set(1, pa.field("customer_id", pa.int64())))
```

```bash
python3 gen_data.py
aws s3 cp out/ s3://${SRC_BUCKET}/hive/ --recursive
aws s3 ls s3://${SRC_BUCKET}/hive/orders/ --recursive --summarize | tail -2
```

（検証後に記載：移行元のファイル数・合計サイズ）

<br>

## Glue テーブルの作成
Athena のクエリエディタで、Hive 外部テーブルを作成してパーティションを登録します。

```sql
CREATE DATABASE IF NOT EXISTS hive_mig_db;

CREATE EXTERNAL TABLE hive_mig_db.orders (
  order_id    bigint,
  customer_id int,
  amount      double,
  status      string,
  ordered_at  timestamp
)
PARTITIONED BY (dt string)
STORED AS PARQUET
LOCATION 's3://hive-mig-src-<アカウントID>/hive/orders/';

MSCK REPAIR TABLE hive_mig_db.orders;
```

`orders_dirty` も、テーブル名と `LOCATION` を `hive/orders_dirty/` に変えて同じように作成します。

作成後、行数を確認しておきます。後で移行先と比較するための基準値です。

```sql
SELECT dt, count(*) AS cnt, sum(amount) AS total
FROM hive_mig_db.orders
GROUP BY dt
ORDER BY dt;
```

（検証後に記載：移行元の行数・合計金額）

<br>

## テーブルバケットと名前空間の作成
移行先のテーブルバケットと名前空間を作成します。移行先テーブルは移行ジョブが作るため、ここでは作りません。

```bash
export TABLE_BUCKET_ARN=$(aws s3tables create-table-bucket \
  --name ${TABLE_BUCKET} \
  --query arn --output text)

aws s3tables create-namespace \
  --table-bucket-arn ${TABLE_BUCKET_ARN} \
  --namespace ${NAMESPACE}
```

コンソールからテーブルバケットを作成すると、AWS 分析サービスとの統合をその場で有効にできます。CLI で作った場合は、コンソールのテーブルバケットの画面から統合を有効にします。

<br>

## 移行用 IAM ロールの作成
移行ジョブは、`migration.s3tables.amazonaws.com` が IAM ロールを引き受けて実行します。
信頼ポリシーでは、引き受けられるのを今回のテーブルバケットへの移行に限定します。

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "migration.s3tables.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "${ACCOUNT_ID}",
          "aws:SourceArn": "${TABLE_BUCKET_ARN}"
        }
      }
    }
  ]
}
EOF
```

権限ポリシーは、ドキュメントの例から KMS 関連のステートメントを除き、今回のリソースに合わせています。Glue テーブルと移行元のプレフィックスは、`orders` と `orders_dirty` の両方を含めるためワイルドカードにしています。

```bash
cat > permission-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSourceTableMetadata",
      "Effect": "Allow",
      "Action": ["glue:GetTable", "glue:GetPartitions", "glue:GetColumnStatisticsForPartition"],
      "Resource": [
        "arn:aws:glue:${AWS_REGION}:${ACCOUNT_ID}:catalog",
        "arn:aws:glue:${AWS_REGION}:${ACCOUNT_ID}:database/${GLUE_DB}",
        "arn:aws:glue:${AWS_REGION}:${ACCOUNT_ID}:table/${GLUE_DB}/*"
      ]
    },
    {
      "Sid": "ListSourceBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::${SRC_BUCKET}"
    },
    {
      "Sid": "ReadSourceObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::${SRC_BUCKET}/hive/*"
    },
    {
      "Sid": "WriteReports",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${REPORT_BUCKET}/migration-reports/*"
    },
    {
      "Sid": "CreateDestinationTable",
      "Effect": "Allow",
      "Action": [
        "s3tables:CreateTable",
        "s3tables:TagResource",
        "s3tables:PutTableEncryption",
        "s3tables:PutTableStorageClass"
      ],
      "Resource": "${TABLE_BUCKET_ARN}"
    },
    {
      "Sid": "WriteDestinationTable",
      "Effect": "Allow",
      "Action": [
        "s3tables:GetTable",
        "s3tables:GetTableMetadataLocation",
        "s3tables:UpdateTableMetadataLocation",
        "s3tables:GetTableData",
        "s3tables:PutTableData",
        "s3tables:PutTableMaintenanceConfiguration"
      ],
      "Resource": "${TABLE_BUCKET_ARN}/table/*"
    }
  ]
}
EOF

export ROLE_ARN=$(aws iam create-role \
  --role-name ${ROLE_NAME} \
  --assume-role-policy-document file://trust-policy.json \
  --query Role.Arn --output text)

aws iam put-role-policy \
  --role-name ${ROLE_NAME} \
  --policy-name S3TablesMigrationPolicy \
  --policy-document file://permission-policy.json
```

移行を実行する作業ユーザー側には、`s3tables:StartTableMigration` などの移行 API の権限と、このロールに対する `iam:PassRole` が必要です。

<br>

# 検証1：Start で移行し、レポートと移行先を確認する
## 確認したいこと
- `--review-migration-plan` を付けると `pendingReview` で止まり、承認するまでデータがコピーされないこと
- 互換性レポートと移行レポートに何が出力されるか
- 移行元の60ファイルが、移行先で何ファイルになるか
- Glue のスキーマ（`int`・`timestamp` など）が Iceberg のどの型に変換されるか
- 移行元と移行先で、行数と合計金額が一致すること

<br>

## 移行の開始
レポートの出力先は検証ごとにプレフィックスを分けておきます。

```bash
export OUTPUT_ORDERS='{"bucket":"arn:aws:s3:::'${REPORT_BUCKET}'","prefixes":{"tableCompatibilityReport":"migration-reports/orders/","objectCompatibilityReport":"migration-reports/orders/","objectMigrationReport":"migration-reports/orders/"}}'

export MIGRATION_ID=$(aws s3tables start-table-migration \
  --role-arn ${ROLE_ARN} \
  --destination-table '{"tableBucketARN":"'${TABLE_BUCKET_ARN}'","namespace":"'${NAMESPACE}'","name":"orders"}' \
  --source-type glueTable \
  --source '{"glueTable":{"catalogId":"'${ACCOUNT_ID}'","databaseName":"'${GLUE_DB}'","tableName":"orders"}}' \
  --migration-output "${OUTPUT_ORDERS}" \
  --review-migration-plan \
  --query migrationId --output text)

echo ${MIGRATION_ID}
```

（検証後に記載：レスポンス）

<br>

## 状態の確認と承認
`get-table-migration` で状態を確認します。`pendingReview` になるまで待ちます。

```bash
aws s3tables get-table-migration --migration-id ${MIGRATION_ID}
```

（検証後に記載：`new` → `planning` → `pendingReview` の遷移と、それぞれの時刻）

`pendingReview` の時点でレポート用バケットを確認し、互換性レポートが出力されていることを確認します。

```bash
aws s3 ls s3://${REPORT_BUCKET}/migration-reports/orders/ --recursive
```

（検証後に記載：出力されたファイルと、table compatibility report / object compatibility report の内容）

この時点で移行先テーブルが作成済みか、作成済みならデータが空かどうかも確認しておきます。

```bash
aws s3tables get-table \
  --table-bucket-arn ${TABLE_BUCKET_ARN} \
  --namespace ${NAMESPACE} \
  --name orders
```

（検証後に記載）

レポートに問題がなければ、状態を `queued` にして承認します。

```bash
aws s3tables update-table-migration \
  --migration-id ${MIGRATION_ID} \
  --requested-state queued
```

`succeeded` になるまで `get-table-migration` で確認します。

（検証後に記載：`queued` → `migrating` → `succeeded` の遷移、承認から完了までの所要時間、レスポンスの統計情報）

<br>

## 移行先の確認
Athena で移行先テーブルの行数と合計金額を集計し、移行元の値と比較します。

```sql
SELECT dt, count(*) AS cnt, sum(amount) AS total
FROM "s3tablescatalog/hive-mig-dest"."analytics"."orders"
GROUP BY dt
ORDER BY dt;
```

（検証後に記載：移行元との比較）

次に、移行先のデータファイル数とサイズを `$files` メタデータテーブルで確認します。

```sql
SELECT partition, count(*) AS files, sum(file_size_in_bytes) AS bytes
FROM "s3tablescatalog/hive-mig-dest"."analytics"."orders$files"
GROUP BY partition;
```

（検証後に記載：移行元60ファイルとの比較、圧縮コーデック変更（snappy → zstd）によるサイズの変化）

最後に、変換後のスキーマとパーティションを確認します。

```sql
SHOW CREATE TABLE "s3tablescatalog/hive-mig-dest"."analytics"."orders";
```

（検証後に記載：Glue の型と Iceberg の型の対応、`dt` がパーティションとして引き継がれているか）

<br>

# 検証2：Sync で移行元の変更を追従する
## 確認したいこと
ドキュメントでは、Sync について次のように説明されています。

- 前回の Start / Sync 以降に変わったオブジェクトとパーティションだけをコピーする
- 移行元で削除されたオブジェクトのデータは、移行先からも削除する
- 前回の Start / Sync 以降に上書きされたオブジェクトは、内容が同じでも失敗する

この3点を、ファイルの追加・削除・上書きの順に1つずつ試します。上書きを最後にするのは、Sync が失敗した後の状態を別に確認するためです。

<br>

## ファイルの追加
新しいパーティション `dt=2026-11-04` にファイルを1つ追加し、パーティションを登録してから Sync します。

追加するファイルは、`gen_data.py` の関数を使って作ります。`order_id` は既存のデータと重複しない値から始めます。

```python:add_data.py
from gen_data import make_rows, write

write("out/orders/dt=2026-11-04/part-000.parquet", make_rows(100001, 500, "2026-11-04"))
```

```bash
python3 add_data.py
aws s3 cp out/orders/dt=2026-11-04/part-000.parquet \
  s3://${SRC_BUCKET}/hive/orders/dt=2026-11-04/part-000.parquet
```

```sql
MSCK REPAIR TABLE hive_mig_db.orders;
```

```bash
export TABLE_ARN=$(aws s3tables get-table \
  --table-bucket-arn ${TABLE_BUCKET_ARN} \
  --namespace ${NAMESPACE} \
  --name orders \
  --query tableARN --output text)

export SYNC_ID=$(aws s3tables sync-table-migration \
  --role-arn ${ROLE_ARN} \
  --table-arn ${TABLE_ARN} \
  --migration-output "${OUTPUT_ORDERS}" \
  --query migrationId --output text)

aws s3tables get-table-migration --migration-id ${SYNC_ID}
```

（検証後に記載：所要時間、統計情報のオブジェクト数が追加分だけになっているか、移行先の行数）

<br>

## ファイルの削除
`dt=2026-11-01` のファイルを1つ削除してから Sync し、移行先の行数が500行減るかを確認します。

```bash
aws s3 rm s3://${SRC_BUCKET}/hive/orders/dt=2026-11-01/part-000.parquet
```

Sync のコマンドは追加のときと同じです。

（検証後に記載：移行先の行数の変化、移行先のスナップショットとファイル数の変化）

<br>

## ファイルの上書き
`dt=2026-11-02` の既存ファイルを、同じ内容のまま同じキーへアップロードし直してから Sync します。

```bash
aws s3 cp out/orders/dt=2026-11-02/part-000.parquet \
  s3://${SRC_BUCKET}/hive/orders/dt=2026-11-02/part-000.parquet
```

（検証後に記載：Sync の最終状態、`failureReason`、object migration report の該当行）

ドキュメントどおりに失敗した場合は、`list-table-migrations` で `hasValidPlan` を確認し、どう復旧できるかを見ます。

```bash
aws s3tables list-table-migrations --table-bucket-arn ${TABLE_BUCKET_ARN}
```

（検証後に記載：復旧方法。上書きしたファイルを別キーにコピーして元を削除すると Sync が通るか、など）

<br>

# 検証3：移行ポリシーで不整合なファイルの扱いを変える
## 確認したいこと
移行ポリシーには2つの設定があり、どちらもデフォルトは `fail` です。Start の時点で決め、後から変えることはできません。

| 設定 | 対象 | 値 |
| --- | --- | --- |
| `unmatchedSourceFieldsPolicy` | 移行先のスキーマにない列を持つファイル | `fail`（ファイルを失敗にする） / `drop`（その列を除いて移行する） |
| `incompatibleValuesPolicy` | 移行先の型に変換できない値を持つ列（nullable のもの） | `fail`（移行を失敗にする） / `setNull`（null で書き込む） |

`orders_dirty` を、デフォルトのポリシーと `drop` / `setNull` の2通りで別々のテーブルに移行し、結果を比べます。
`extra_column.parquet` は `unmatchedSourceFieldsPolicy` の、`type_mismatch.parquet` は `incompatibleValuesPolicy` の挙動を見るためのファイルです。

ただし、`int64` を `int` として読む変換が「変換できない値」に当たるのかは、ドキュメントの例（`bigint` のファイルを `int` 列に移す例）から読み取った想定です。値の範囲が `int` に収まる場合に変換されるのか、ファイル単位で非互換と判定されるのかも、あわせて確認します。

<br>

## デフォルトのポリシー（fail / fail）で移行する

```bash
export OUTPUT_DIRTY='{"bucket":"arn:aws:s3:::'${REPORT_BUCKET}'","prefixes":{"tableCompatibilityReport":"migration-reports/dirty-default/","objectCompatibilityReport":"migration-reports/dirty-default/","objectMigrationReport":"migration-reports/dirty-default/"}}'

export DIRTY_DEFAULT_ID=$(aws s3tables start-table-migration \
  --role-arn ${ROLE_ARN} \
  --destination-table '{"tableBucketARN":"'${TABLE_BUCKET_ARN}'","namespace":"'${NAMESPACE}'","name":"orders_dirty_default"}' \
  --source-type glueTable \
  --source '{"glueTable":{"catalogId":"'${ACCOUNT_ID}'","databaseName":"'${GLUE_DB}'","tableName":"orders_dirty"}}' \
  --migration-output "${OUTPUT_DIRTY}" \
  --query migrationId --output text)

aws s3tables get-table-migration --migration-id ${DIRTY_DEFAULT_ID}
```

（検証後に記載：最終状態、`failureReason`、互換性レポートでの検出結果、失敗時に移行先へコミット済みのデータが残っているか）

<br>

## drop / setNull で移行する

```bash
export OUTPUT_DIRTY2='{"bucket":"arn:aws:s3:::'${REPORT_BUCKET}'","prefixes":{"tableCompatibilityReport":"migration-reports/dirty-relaxed/","objectCompatibilityReport":"migration-reports/dirty-relaxed/","objectMigrationReport":"migration-reports/dirty-relaxed/"}}'

export DIRTY_RELAXED_ID=$(aws s3tables start-table-migration \
  --role-arn ${ROLE_ARN} \
  --destination-table '{"tableBucketARN":"'${TABLE_BUCKET_ARN}'","namespace":"'${NAMESPACE}'","name":"orders_dirty_relaxed"}' \
  --source-type glueTable \
  --source '{"glueTable":{"catalogId":"'${ACCOUNT_ID}'","databaseName":"'${GLUE_DB}'","tableName":"orders_dirty"}}' \
  --data-migration '{"migrationPolicies":{"unmatchedSourceFieldsPolicy":"drop","incompatibleValuesPolicy":"setNull"}}' \
  --migration-output "${OUTPUT_DIRTY2}" \
  --query migrationId --output text)

aws s3tables get-table-migration --migration-id ${DIRTY_RELAXED_ID}
```

移行後、`type_mismatch.parquet` 由来の行で `customer_id` が null になっているかを確認します。`gen_data.py` では `type_mismatch.parquet` に `order_id` 33001〜33500 を割り当てているため、この範囲で行を特定できます。

```sql
SELECT
  order_id BETWEEN 33001 AND 33500 AS from_type_mismatch,
  count(*) AS total,
  count(customer_id) AS non_null_customer
FROM "s3tablescatalog/hive-mig-dest"."analytics"."orders_dirty_relaxed"
GROUP BY 1;
```

（検証後に記載：最終状態、行数、null になった行数、`audit` 列が移行先に存在しないこと、object migration report の記載）

<br>

## 結果の比較
（検証後に記載：2通りのポリシーの結果を表にまとめる）

| ファイル | fail / fail | drop / setNull |
| --- | --- | --- |
| 正常ファイル（5個） | | |
| `extra_column.parquet` | | |
| `type_mismatch.parquet` | | |

<br>

# 検証4：Complete で書き込みを有効にする
## 確認したいこと
- Complete の前は、移行先テーブルへの `INSERT` が失敗すること
- Complete の後は `INSERT` が成功し、コンパクションなどのメンテナンスが有効になること
- Complete の後は Sync ができなくなること

<br>

## Complete 前の書き込み
Athena から移行先へ1行追加してみます。

```sql
INSERT INTO "s3tablescatalog/hive-mig-dest"."analytics"."orders"
VALUES (999999, 1, 100.0, 'created', TIMESTAMP '2026-11-04 00:00:00', '2026-11-04');
```

（検証後に記載：エラーメッセージ）

<br>

## Complete の実行
Complete は取り消せないため、検証1〜2で移行先の内容を確認し終えてから実行します。

```bash
aws s3tables complete-table-migration --table-arn ${TABLE_ARN}
```

実行後、テーブルの情報とメンテナンス設定を確認します。

```bash
aws s3tables get-table --table-arn ${TABLE_ARN}
aws s3tables get-table-maintenance-configuration \
  --table-bucket-arn ${TABLE_BUCKET_ARN} \
  --namespace ${NAMESPACE} \
  --name orders
```

（検証後に記載：Complete 前後での `get-table` の差分、メンテナンス設定の値）

<br>

## Complete 後の書き込みと Sync
同じ `INSERT` をもう一度実行します。

（検証後に記載：結果）

Sync も実行してみます。

```bash
aws s3tables sync-table-migration \
  --role-arn ${ROLE_ARN} \
  --table-arn ${TABLE_ARN} \
  --migration-output "${OUTPUT_ORDERS}"
```

（検証後に記載：エラーメッセージ）

<br>

# 後片付け
作成したリソースを、依存関係の逆順に削除します。

```bash
for t in orders orders_dirty_default orders_dirty_relaxed; do
  aws s3tables delete-table --table-bucket-arn ${TABLE_BUCKET_ARN} --namespace ${NAMESPACE} --name ${t}
done
aws s3tables delete-namespace --table-bucket-arn ${TABLE_BUCKET_ARN} --namespace ${NAMESPACE}
aws s3tables delete-table-bucket --table-bucket-arn ${TABLE_BUCKET_ARN}

aws glue delete-table --database-name ${GLUE_DB} --name orders
aws glue delete-table --database-name ${GLUE_DB} --name orders_dirty
aws glue delete-database --name ${GLUE_DB}

aws s3 rb s3://${SRC_BUCKET} --force
aws s3 rb s3://${REPORT_BUCKET} --force

aws iam delete-role-policy --role-name ${ROLE_NAME} --policy-name S3TablesMigrationPolicy
aws iam delete-role --role-name ${ROLE_NAME}
```

Lake Formation で権限を付与した場合は、その権限も取り消します。

<br>

# おわりに
（検証後に記載）

- 移行全体の手間（ロールとレポート用バケットを用意すれば、あとは API 呼び出しだけで済んだか）
- 検証で分かった注意点（Sync の上書き、移行ポリシー、型変換など）
- 実運用で移行するときの進め方（まず `--review-migration-plan` でレポートを確認する、Sync で差分を追従してから書き込みを止めて Complete する、など）

<br>

# 参考リンク
- [Amazon S3 Tables now offer managed migration of Apache Hive tables to Apache Iceberg（AWS ブログ）](https://aws.amazon.com/jp/blogs/news/amazon-s3-tables-now-offer-managed-migration-of-apache-hive-tables-to-apache-iceberg/)
- [Migrating tables to S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-migrate.html)
- [How to migrate a table to S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-migrate-how-it-works.html)
- [Setting up table migration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-migrate-setting-up.html)
- [Running a table migration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-migrate-running.html)
- [[アップデート] Amazon S3 Tables で Glue の Hive テーブルを Iceberg テーブルへマネージド移行できるようになったので試してみた（DevelopersIO）](https://dev.classmethod.jp/articles/20261105-amazon-s3-tables-migration-hive-to-iceberg/)
