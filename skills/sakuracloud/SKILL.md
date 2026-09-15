---
name: sakuracloud
description: さくらのクラウド・高火力シリーズで GPU 計算を行うときに使う。高火力DOK（コンテナタスク投入、Web API、プラン v100-32gb/h100-80gb）、高火力VRT / GPUプラン（H100・V100 の GPU サーバを usacloud で作成して SSH）、オブジェクトストレージ（S3互換、s3.isk01.sakurastorage.jp）、APIキー・ゾーン（is1a/tk1a 等）、NVIDIA ドライバ/CUDA 導入、高火力PHY。「さくら」「sakura」「高火力」「DOK」「VRT」「usacloud」「さくらでGPU」「sacloud」等が出てきたら参照。詳細は references/handbook.md。
---

# さくらのクラウド / 高火力

さくらインターネットの GPU クラウド。**ジョブスケジューラは存在しない**（`qsub` / `sbatch` は無い）。
計算資源は「API でコンテナタスクを投げる」か「VM を自分で立てて SSH する」かのどちらか。

詳細リファレンス → `references/handbook.md`

## どれを使うか

| 製品 | 形態 | 課金 | 向いている用途 |
|---|---|---|---|
| **高火力DOK** | コンテナタスク | 実行秒課金 | 単発の学習・推論ジョブ。`qsub` に一番近い |
| **高火力VRT**（さくらのクラウド GPUプラン） | GPU 付き VM | 時間/日/月割 | SSH して対話的に開発、中期間の学習 |
| **高火力PHY** | ベアメタル専有 | 最低 2 か月〜 | 複数筐体をつないだ大規模学習 |

迷ったら DOK。VM を管理したくなければ DOK 一択。

## 共通セットアップ

APIキー（アクセストークン / シークレット）はコントロールパネルの「APIキー管理」で発行する。

```bash
export SAKURACLOUD_ACCESS_TOKEN="<アクセストークン>"
export SAKURACLOUD_ACCESS_TOKEN_SECRET="<シークレット>"
export SAKURACLOUD_ZONE="is1a"
```

ゾーン: `is1a`(石狩第1) / `is1b`(石狩第2) / `is1c`(石狩第3) / `tk1a`(東京第1) / `tk1b`(東京第2) / `tk1v`(Sandbox)

**GPU は `is1a` 限定**（VRT の H100/V100 プラン、DOK の API エンドポイントとも `is1a` のみ）。
API はゾーンごとに独立しており、リソースはゾーンを跨げない。

### usacloud（公式 CLI）

```bash
curl -fsSL https://github.com/sacloud/usacloud/releases/latest/download/install.sh | bash   # macOS/Linux
brew tap sacloud/usacloud && brew install --cask usacloud                                   # Homebrew

usacloud config create --name default --token <トークン> --secret <シークレット> --zone is1a --use
```

設定は `~/.usacloud/<プロファイル名>/config.json` に保存される。

> **DOK は usacloud では操作できない。** 専用 CLI も存在しない（Web API とコントロールパネルのみ）。

---

## 高火力DOK（コンテナタスク）

### タスク投入

Base URL: `https://secure.sakura.ad.jp/cloud/zone/is1a/api/managed-container/1.0`
認証は Basic 認証（ユーザ名=アクセストークン、パスワード=シークレット）。

```bash
DOK="https://secure.sakura.ad.jp/cloud/zone/is1a/api/managed-container/1.0"
AUTH="${SAKURACLOUD_ACCESS_TOKEN}:${SAKURACLOUD_ACCESS_TOKEN_SECRET}"

# タスク作成
curl -u "$AUTH" -X POST "$DOK/tasks/" -H 'Content-Type: application/json' -d '{
  "name": "train-job",
  "containers": [{
    "image": "example.sakuracr.jp/train:latest",
    "registry": "<レジストリ認証情報のUUID>",
    "entrypoint": ["sh", "-c"],
    "command": ["python train.py"],
    "environment": {
      "AWS_ACCESS_KEY_ID": "...",
      "AWS_SECRET_ACCESS_KEY": "...",
      "AWS_DEFAULT_REGION": "jp-north-1",
      "AWS_ENDPOINT_URL": "https://s3.isk01.sakurastorage.jp",
      "S3_BUCKET": "my-bucket"
    },
    "plan": "h100-80gb"
  }],
  "tags": ["experiment"],
  "execution_time_limit_sec": 86400
}'

curl -u "$AUTH" -X GET    "$DOK/tasks/"                    # 一覧
curl -u "$AUTH" -X GET    "$DOK/tasks/<taskId>/"           # 詳細・ステータス
curl -u "$AUTH" -X POST   "$DOK/tasks/<taskId>/cancel/"    # 実行中タスクの中断
curl -u "$AUTH" -X DELETE "$DOK/tasks/<taskId>/"           # 削除（中断ではない）
```

`containers[]` の必須フィールドは `image` / `command` / `entrypoint` / `plan` の 4 つ。
`registry` はパブリックイメージなら省略可。

| フィールド | 説明 |
|---|---|
| `name` | タスク名（必須） |
| `execution_time_limit_sec` | 実行時間上限（秒）。600 以上。`null` で無制限。省略時 1728000（20 日） |
| `environment` | 環境変数のマップ。キー+値の合計 8192 文字未満 |
| `http` | `{"port": 80, "path": "/"}` でリッスンポートを公開（HTTPS のみ） |
| `ssh` | `{"shell": "/bin/bash"}`（`/bin/sh` `/bin/bash` `/bin/zsh`） |

### プラン

| プラン名 | GPU | コア | メモリ | 最低利用時間 |
|---|---|---|---|---|
| `v100-32gb` | V100 32GB × 1 | 3 | 40GB | 1 秒 |
| `h100-80gb` | H100 80GB × 1 | 10 | 192GB | 60 秒 |
| `h100-8gpu-80gb` | H100 80GB × 8 | 160 | 1760GB | 60 秒 |

`v100-32gb` は **2027-03-31 提供終了予定**（新規受付終了 2027-02-28）。新規は H100 系を選ぶ。

### コンテナ側の作法

タスク内で使える定義済み環境変数:

| 変数 | 内容 |
|---|---|
| `SAKURA_TASK_ID` | タスクを一意に識別する ID |
| `SAKURA_ARTIFACT_DIR` | アーティファクト保存先（例: `/opt/artifact`）。ここに置いたファイルがタスク終了後に tar.gz でダウンロードできる |

- **`SAKURA_` で始まる名前の環境変数は自前で定義できない。**
- プラットフォームは `linux/amd64` 固定。Apple Silicon でビルドするなら `--platform` 指定が必須。
- 学習データはオブジェクトストレージから取り、結果は `$SAKURA_ARTIFACT_DIR` に書く、が定石。

```bash
# イメージのビルドと push（レジストリは事前にコンパネで作成）
docker login example.sakuracr.jp
docker buildx build --platform linux/amd64 -t example.sakuracr.jp/train:latest --push .
```

---

## 高火力VRT / GPUプラン（GPU 付き VM）

### プラン

| プラン | 仮想コア | メモリ | GPU | 備考 |
|---|---|---|---|---|
| 24Core-240GB-H100x1 | 24 | 240GB | H100 × 1 | 約 6.9TiB の NVMe 付属 |
| 4Core-56GB-V100x1 | 4 | 56GB | V100 × 1 | **2027-03-31 提供終了予定** |

**石狩第1ゾーン（`is1a`）のみ**。Windows Server 不可、専有ホスト上では利用不可。

### サーバ作成

```bash
usacloud server-plan list --zone is1a          # プラン確認
usacloud ssh-key create --name mykey --public-key ~/.ssh/id_ed25519.pub

usacloud server create \
  --zone is1a --name gpu01 \
  --cpu 24 --memory 240 --gpu 1 \
  --disk-disk-plan ssd --disk-size 100 \
  --disk-os-type ubuntu \
  --disk-edit-ssh-key-ids <SSH鍵ID> \
  --disk-edit-disable-pw-auth \
  --boot-after-create

# --disk-os-type に指定できる値は `usacloud server create --help` で確認する
# （アーカイブを直接指定するなら --disk-source-archive-id <ID>）

usacloud server list --zone is1a
usacloud server ssh gpu01 --zone is1a -l ubuntu -i ~/.ssh/id_ed25519
usacloud server shutdown gpu01 --zone is1a     # 課金停止はここ
usacloud server delete gpu01 --zone is1a
```

`--parameters '<JSON>'` でまとめて渡すこともできる。`--generate-skeleton` で雛形が出る。

> `--cpu` / `--memory` / `--gpu` の組み合わせでプランが決まる。公式ドキュメントに VRT プランと
> これらの値の対応を明示した実例は無いので、`server-plan list` の出力で実在するプランを確認してから指定する。

### NVIDIA ドライバ / CUDA

**プリインストールされていない。** サーバ作成後に自分で入れる。

Ubuntu 24.04:

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-ubuntu2404.pin
sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/12.9.1/local_installers/cuda-repo-ubuntu2404-12-9-local_12.9.1-575.57.08-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2404-12-9-local_12.9.1-575.57.08-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2404-12-9-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get install -y cuda-toolkit-12-9 cuda-drivers
```

AlmaLinux 9 / Rocky Linux 9:

```bash
sudo dnf install -y wget kernel-devel-matched kernel-headers
sudo dnf config-manager --set-enabled crb
sudo dnf install -y epel-release
wget https://developer.download.nvidia.com/compute/cuda/12.9.1/local_installers/cuda-repo-rhel9-12-9-local-12.9.1_575.57.08-1.x86_64.rpm
sudo rpm -i cuda-repo-rhel9-12-9-local-12.9.1_575.57.08-1.x86_64.rpm
sudo dnf clean all
sudo dnf install -y cuda-toolkit-12-9
sudo dnf module install -y nvidia-driver:latest-dkms
```

確認: `nvidia-smi` / `lspci | grep -i nvidia`

CUDA のバージョンは変わるので、最新は https://developer.nvidia.com/cuda-downloads と
https://manual.sakura.ad.jp/cloud/server/gpu-plan.html を確認する。

### 付属 NVMe（H100 プラン）

```bash
sudo apt-get install -y nvme-cli
nvme list
sudo mkfs -t xfs /dev/nvme0n1
sudo mkdir -p /mnt/nvme && sudo mount /dev/nvme0n1 /mnt/nvme
```

**停電で内容が消失する。** スクラッチ専用。チェックポイントはオブジェクトストレージへ退避する。

---

## オブジェクトストレージ（S3 互換）

| サイト | エンドポイント | リージョン名 |
|---|---|---|
| 石狩第1 | `s3.isk01.sakurastorage.jp` | `jp-north-1` |
| 東京第1 | `s3.tky01.sakurastorage.jp` | `jp-east-1` |

署名バージョンは `s3v4`。バケットとアクセスキーはコントロールパネルの「バケット」「パーミッション」で作る
（アクセスキーは 1 パーミッションにつき 1 個まで）。

```bash
export AWS_ACCESS_KEY_ID="..." AWS_SECRET_ACCESS_KEY="..." AWS_DEFAULT_REGION="jp-north-1"
EP="https://s3.isk01.sakurastorage.jp"

aws --endpoint-url="$EP" s3 ls
aws --endpoint-url="$EP" s3 cp ./dataset.tar s3://my-bucket/
aws --endpoint-url="$EP" s3 ls s3://my-bucket/ --summarize --recursive
aws --endpoint-url="$EP" s3 rm s3://my-bucket/old.ckpt

# 失敗したマルチパートアップロードの確認・中断（課金され続けるので要チェック）
aws s3api list-multipart-uploads --bucket my-bucket --endpoint-url="$EP" --output json
aws s3api abort-multipart-upload --bucket my-bucket --upload-id "..." --key "file.dat" --endpoint-url="$EP"
```

制限: 1 プロジェクト 20 バケット / 1 バケット 10TiB / 1,000 万オブジェクト / 1 オブジェクト 5TiB。

---

## 課金の考え方

- **時間割 10 時間 = 日割 1 日分**、**日割 20 日分 = 月額 1 か月分**。合算額と月額の**安い方**が採用される。
- サーバは**起動で課金開始・停止で課金終了**。ディスクは**作成から削除まで**課金（サーバを停めてもディスク代は止まらない）。
- DOK は `プラン単価 × 実行秒数`（最低利用時間あり）。
- データ転送量による従量課金はなし。

金額は変動するので https://cloud.sakura.ad.jp/payment/simulation/ で確認する。

## よくあるハマりどころ

| 症状 | 原因・対処 |
|---|---|
| GPU プランが選べない | ゾーンが `is1a` 以外。GPU は石狩第1のみ |
| `usacloud` で DOK を操作できない | DOK は usacloud 非対応。Web API かコンパネを使う |
| DOK タスクが待機したまま進まない | 1 プロジェクトにつき同時 1 タスクまで。先行タスクの終了待ち |
| DOK タスクが 20 日で切れた | `execution_time_limit_sec` の既定値。明示指定するか `null` で無制限に |
| DOK でイメージが動かない | `linux/amd64` でビルドしているか確認（`docker buildx build --platform linux/amd64`） |
| 成果物が取れない | `$SAKURA_ARTIFACT_DIR` に書いているか。ダウンロード期限は **72 時間** |
| 環境変数が無視される | `SAKURA_` で始まる名前は使用不可 |
| `nvidia-smi` が無い | ドライバは未導入。自分でインストールする |
| 再起動したら NVMe のデータが消えた | 付属 NVMe は揮発する。永続データを置かない |
| サーバを停めたのに課金が続く | ディスク・グローバル IP は別課金。IP は月額固定で日割なし |
| SSH できない | パケットフィルタのルール順（上から評価、最大 30 ルール）を確認 |
| DHCP が効かなくなった | パケットフィルタで 67/UDP・68/UDP を塞いでいる |
| オブジェクトストレージの使用量が減らない | 未完了のマルチパートアップロードが残っている。`list-multipart-uploads` で確認 |
