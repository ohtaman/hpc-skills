# さくらのクラウド / 高火力 詳細リファレンス

出典:
- さくらのクラウド マニュアル https://manual.sakura.ad.jp/cloud/
- 高火力DOK マニュアル https://manual.sakura.ad.jp/cloud/manual-koukaryoku-container.html
- 高火力DOK API 仕様 https://manual.sakura.ad.jp/koukaryoku-dok-api/spec.html
- usacloud ドキュメント https://docs.usacloud.jp/usacloud/
- さくらのクラウド API https://manual.sakura.ad.jp/cloud-api/1.1/

---

## サービスの使い分け

さくらの GPU サービス「高火力」は提供形態の異なる 3 製品ライン。

| 製品 | 形態 | 課金 | 向いている用途 | HPC 的な位置づけ |
|---|---|---|---|---|
| **高火力DOK** | コンテナタスク | 実行秒課金 | 単発の学習・推論ジョブ、バッチ処理 | `qsub` に一番近い |
| **高火力VRT**（= さくらのクラウド GPUプラン） | GPU 付き VM | 時間/日/月割 | SSH して対話的に開発、中期間の学習 | 占有ノードを借りる感覚 |
| **高火力PHY** | ベアメタル専有サーバ | 最低 2 か月〜 | 複数筐体をつないだ大規模学習 | 設備調達に近い |

- 認証はいずれも共通で、さくらのクラウドの **APIキー**（アクセストークン / アクセストークンシークレット）を使う。
- 複数筐体接続が必要な大規模学習は PHY、短期利用は VRT、という案内が公式サイトにある。
  （出典: https://cloud.sakura.ad.jp/products/server/gpu/ ）

---

## 共通セットアップ

### ゾーン

| ゾーンID | 名称 |
|---|---|
| `is1a` | 石狩第1 |
| `is1b` | 石狩第2 |
| `is1c` | 石狩第3 |
| `tk1a` | 東京第1 |
| `tk1b` | 東京第2 |
| `tk1v` | Sandbox（検証用） |

**API はゾーンごとに存在する**。第2ゾーンにサーバを作るときは第2ゾーンの API を叩く必要がある。
リソースはゾーンを跨げない（別ゾーンのディスクをサーバに接続する、等は不可）。

API エンドポイント形式:

```
https://secure.sakura.ad.jp/cloud/zone/{zone_id}/api/cloud/1.1/
```

（出典: https://manual.sakura.ad.jp/cloud-api/1.1/ ）

### APIキー

コントロールパネルの「APIキー管理」でアクセストークン / アクセストークンシークレットを発行する。

### 環境変数（usacloud / Terraform 共通）

| 環境変数 | 用途 |
|---|---|
| `SAKURACLOUD_ACCESS_TOKEN` | アクセストークン |
| `SAKURACLOUD_ACCESS_TOKEN_SECRET` | アクセストークンシークレット |
| `SAKURACLOUD_ZONE` | 対象ゾーン |
| `SAKURACLOUD_DEFAULT_ZONE` | デフォルトゾーン |
| `SAKURACLOUD_PROFILE` | 使用するプロファイル名 |
| `SAKURACLOUD_PROFILE_DIR` | プロファイル保存先 |
| `SAKURACLOUD_API_ROOT_URL` | API のルート URL（上書き用） |
| `SAKURACLOUD_DEFAULT_OUTPUT_TYPE` | 既定の出力形式 |
| `SAKURACLOUD_RETRY_MAX` | リトライ回数上限 |
| `SAKURACLOUD_API_REQUEST_TIMEOUT` | リクエストタイムアウト |
| `SAKURACLOUD_API_REQUEST_RATE_LIMIT` | リクエストレート上限 |

（出典: https://docs.usacloud.jp/usacloud/references/env/ ）

---

### usacloud（公式 CLI）

さくらのクラウド用の公式 CLI。**高火力DOK には対応していない**（DOK は Web API とコントロールパネルのみ。
専用 CLI も sacloud の GitHub org には存在しない）。

インストール:

```bash
# macOS / Linux
curl -fsSL https://github.com/sacloud/usacloud/releases/latest/download/install.sh | bash

# Homebrew
brew tap sacloud/usacloud
brew install --cask usacloud

# Windows (Chocolatey)
choco install usacloud

# Docker
docker run ghcr.io/sacloud/usacloud ...
```

初期設定（対話形式なら `usacloud config` だけでよい）:

```bash
usacloud config create --name default \
  --token <アクセストークン> --secret <シークレット> \
  --zone is1a --default-output-type json --use
```

`config create` のフラグ:

```
--access-token string        (aliases: --token)
--access-token-secret string (aliases: --secret)
--default-output-type string
--name string
--no-color
--use
--zone string
```

設定は `~/.usacloud/<プロファイル名>/config.json` に保存される。
（出典: https://docs.usacloud.jp/usacloud/references/config/ , https://docs.usacloud.jp/usacloud/installation/start_guide/ ）

#### 主なリソースとサブコマンド

| リソース | 代表サブコマンド |
|---|---|
| `server` | list / create / read / update / delete / boot / shutdown / reset / send-nmi / ssh / vnc / rdp / monitor-cpu / wait-until-ready / wait-until-shutdown |
| `disk` | list / create / read / update / delete / connect-to-server / disconnect-from-server / edit / resize-partition / monitor-disk / wait-until-ready |
| `ssh-key` | list / create / generate / read / update / delete |
| `server-plan` | list（フィルタは `--names` のみ。`--zone` 必須） |
| `switch` / `internet` / `packet-filter` | ネットワーク系 |

（出典: https://docs.usacloud.jp/usacloud/references/ 配下）

#### `server create` の主要フラグ（逐語）

```
=== Plan options ===
    --cpu int             (*required) (aliases: --core) (default 1)
    --memory int          (*required) (default 1)
    --commitment string   (*required) options: [standard/dedicatedcpu] (default "standard")
    --gpu int
    --generation string   (*required) options: [default/g100/g200] (default "default")

=== Server-specific options ===
    --boot-after-create
    --cdrom-id int              (aliases: --iso-image-id)
    --interface-driver string   (*required) options: [virtio/e1000] (default "virtio")
    --private-host-id int

=== Disk options ===
    --disk-connection string    options: [virtio/ide]
    --disk-disk-plan string     options: [ssd/hdd]
    --disk-os-type string
    --disk-size int             (aliases: --size-gb)
    --disk-source-archive-id int
    --disk-source-disk-id int

=== Edit disk options ===
    --disk-edit-host-name string
    --disk-edit-password string
    --disk-edit-ip-address string
    --disk-edit-netmask int        (aliases: --network-mask-len)
    --disk-edit-gateway string     (aliases: --default-route)
    --disk-edit-disable-pw-auth
    --disk-edit-enable-dhcp
    --disk-edit-ssh-keys strings
    --disk-edit-ssh-key-ids int

=== Network options ===
    --network-interface-packet-filter-id int
    --network-interface-upstream string   options: [shared/disconnected/(switch-id)]
    --network-interface-user-ip-address string

=== Zone options ===
    --zone string   (*required)

=== Input options ===
-y, --assumeyes
    --generate-skeleton
    --parameters string
```

`--parameters` 用 JSON の雛形（`--generate-skeleton` で生成できる）:

```json
{
    "Zone": "tk1a | tk1b | is1a | is1b | tk1v",
    "Name": "example",
    "CPU": 1,
    "Memory": 2,
    "GPU": 0,
    "Commitment": "standard | dedicatedcpu",
    "Generation": "default | g100 | g200",
    "InterfaceDriver": "virtio | e1000",
    "BootAfterCreate": true,
    "NetworkInterfaces": [{"Upstream": "shared | disconnected | (switch-id)"}],
    "Disks": [{
        "DiskPlan": "ssd | hdd",
        "SourceArchiveID": 123456789012,
        "SizeGB": 20,
        "OSType": "...",
        "EditDisk": {
            "HostName": "hostname",
            "Password": "password",
            "SSHKeys": ["/path/to/your/public/key", "ssh-rsa ..."],
            "SSHKeyIDs": [123456789012],
            "IsSSHKeysEphemeral": true
        }
    }]
}
```

**GPU プランの指定方法についての注意**: `server create` / `server update` には `--gpu int`（GPU 個数）が
あるが、「GPU プラン名」を直接指定するフラグは無い。`--cpu` / `--memory` / `--gpu` の組み合わせで決まる。
公式ドキュメントに VRT プラン（V100=4core/56GB、H100=24core/240GB）と usacloud のフラグ値を
対応づけた実行例は**掲載されていない**ため、`usacloud server-plan list --zone is1a` で実在するプランを
確認してから指定すること。

#### `server ssh` のフラグ（逐語）

```
ssh { ID | NAME | TAG } [flags]

=== Server-specific options ===
-i, --key string
    --password string    (aliases: --pass-phrase)
-p, --port int           (*required) (default 22)
-l, --user string
    --wait-until-ready   (aliases: --wait)

=== Zone options ===
    --zone string   (*required)
```

#### SSH 鍵の登録

```
usacloud ssh-key create --name <name> --public-key <公開鍵の中身 または パス>
```

作成された SSH 鍵の ID を、サーバ作成時に `--disk-edit-ssh-key-ids`（JSON では `SSHKeyIDs`）で注入する。
（出典: https://docs.usacloud.jp/usacloud/references/ssh-key/ ）

---

## 高火力VRT / さくらのクラウド GPUプラン

出典: https://manual.sakura.ad.jp/cloud/server/gpu-plan.html

### プラン

| プラン | 仮想コア | メモリ | GPU | 備考 |
|---|---|---|---|---|
| NVIDIA H100 | 24 | 240GB | H100 × 1（専有） | 約 6.9TiB の NVMe が付属 |
| NVIDIA V100 | 4 | 56GB | V100 × 1（専有） | **2027-03-31 提供終了予定** |

- **利用可能ゾーンは石狩第1（`is1a`）のみ**。
- 対応 OS:
  - H100: Ubuntu Server 24.04 / 22.04、AlmaLinux 10.1 / 10.0 / 9.7 / 8.10 / 8.9、Rocky Linux 10.1 / 10.0 / 9.7 / 8.10 / 8.9（いずれも 64bit）
  - V100: Ubuntu Server 24.04 / 22.04 のみ
- Windows Server プランは選択不可。
- 専有ホスト上では利用不可。
- 第三者に利用させる場合は事前承諾が必要。

### 公式プラン名

- `高火力 VRT / 4Core-56GB-V100x1 プラン`
- `高火力 VRT / 24Core-240GB-H100x1 プラン`

> 「現在、高火力 VRT / 4Core-56GB-V100x1 プラン、高火力 VRT / 24Core-240GB-H100x1 プラン、
> どちらも石狩第1ゾーンのみでの提供となります。」

### NVIDIA ドライバ / CUDA のインストール

公式マニュアルが案内している手順（CUDA 12.9.1 / ドライバ 575.57.08 時点）。

Ubuntu 24.04:

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-ubuntu2404.pin
sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/12.9.1/local_installers/cuda-repo-ubuntu2404-12-9-local_12.9.1-575.57.08-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2404-12-9-local_12.9.1-575.57.08-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2404-12-9-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get install -y cuda-toolkit-12-9
sudo apt-get install -y cuda-drivers
```

AlmaLinux 9.5 / Rocky Linux 9.5:

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

動作確認:

```bash
nvidia-smi
lspci | grep -i nvidia
```

CUDA のバージョンは更新されるので、実際に入れる前に
https://developer.nvidia.com/cuda-downloads と
https://manual.sakura.ad.jp/cloud/server/gpu-plan.html を確認すること。

### 付属 NVMe の初期化（H100 プラン）

```bash
sudo apt-get install -y nvme-cli
nvme list
sudo mkfs -t xfs /dev/nvme0n1
sudo mkdir -p /mnt/nvme
sudo mount /dev/nvme0n1 /mnt/nvme
```

### 注意点

- 付属 NVMe（約 6.9TiB）は**停電で内容が消失する**。永続データを置かない。
- 短時間での起動 / 停止の繰り返しは避ける。
- NVIDIA ドライバはプリインストールされておらず、サーバ作成後に自分で入れる。

---

## 高火力DOK

出典: https://manual.sakura.ad.jp/cloud/manual-koukaryoku-container.html

### プラン

| プラン名 | GPU | コア | メモリ | 帯域 | 最低利用時間 |
|---|---|---|---|---|---|
| `v100-32gb` | V100 32GB × 1 | 3 | 40GB | 200Mbps | 1 秒 |
| `h100-80gb` | H100 80GB × 1 | 10 | 192GB | 250Mbps | 60 秒 |
| `h100-8gpu-80gb` | H100 80GB × 8（計 640GB） | 160 | 1760GB | 250Mbps | 60 秒 |

- `SAKURA_ARTIFACT_DIR` の容量は全プラン 20GB。
- V100 プランは **2027-03-31 提供終了予定**（新規タスク受付終了 2027-02-28）。
  （出典: https://www.sakura.ad.jp/corporate/information/announcements/2026/06/30/1968225063/ ）

### Web API

**高火力DOK に CLI は無い**（sacloud の GitHub org にも DOK 用 CLI は存在せず、usacloud にも
対応リソースが無い）。操作は Web API かコントロールパネルで行う。

- Base URL: `https://secure.sakura.ad.jp/cloud/zone/is1a/api/managed-container/1.0`
- 認証: Basic 認証（ユーザ名 = アクセストークン、パスワード = アクセストークンシークレット）
- API 仕様の `servers` は `is1a` の 1 件のみ。**対応ゾーンは石狩第1のみ**

| 操作 | メソッド・パス |
|---|---|
| タスク作成 | `POST /tasks/` |
| タスク一覧 | `GET /tasks/` |
| タスク詳細 | `GET /tasks/{taskId}/` |
| タスク中断 | `POST /tasks/{taskId}/cancel/` |
| タスク削除 | `DELETE /tasks/{taskId}/`（実行中タスクのキャンセル API **ではない**） |

#### リクエストボディ

`CreateTaskRequest`（必須: `name`, `containers`）

| フィールド | 型 | 説明 |
|---|---|---|
| `name` | string | タスク名（必須） |
| `containers` | array | コンテナ定義の配列（必須） |
| `tags` | array[string] | 任意のタグ |
| `execution_time_limit_sec` | integer / null | 実行時間上限（秒）。**600 以上**。`null` で無制限。省略時 **1728000（20 日）** |

`ContainerDefinition`（必須: `image`, `command`, `entrypoint`, `plan`）

| フィールド | 型 | 説明 |
|---|---|---|
| `image` | string | 例 `nginx:latest` |
| `registry` | uuid / null | コンテナレジストリー認証情報の UUID。パブリックイメージなら不要 |
| `command` | array[string] | 例 `["/bin/sh","-c","env"]` |
| `entrypoint` | array[string] | 例 `["sh","-c"]` |
| `environment` | object | 環境変数のマップ。**キーと値の合計が 8192 文字未満** |
| `http` | object / null | `{"port": 80, "path": "/"}` |
| `ssh` | object / null | `{"shell": "/bin/bash"}`（`/bin/sh` `/bin/bash` `/bin/zsh`） |
| `plan` | enum | `v100-32gb` / `h100-80gb`（`h100-8gpu-80gb` は β 扱いで OpenAPI の enum には未反映） |

（出典: https://manual.sakura.ad.jp/koukaryoku-dok-api/spec.html ）

### 定義済み環境変数

| 変数名 | 意味 | 値の例 |
|---|---|---|
| `SAKURA_TASK_ID` | タスクを一意に識別する ID | `be7fdd79-b5e6-4654-a08b-d4143c5d0351` |
| `SAKURA_ARTIFACT_DIR` | アーティファクト保存用ディレクトリ。ここに保存したファイルがタスク終了後に tar.gz 形式でダウンロードできる | `/opt/artifact` |

`SAKURA_` で始まる名前の環境変数は自分で定義できない。
（出典: https://manual.sakura.ad.jp/cloud/koukaryoku-container/running-tasks.html ）

### コンテナレジストリ連携

要件は「インターネットからアクセス可能」「Docker Registry HTTP API V2 に準拠」の 2 点のみ。
さくらのクラウドの「コンテナレジストリ」アプライアンスのほか、Docker Hub / ECR などのプライベート
レジストリも使える。

```bash
docker login {コンテナレジストリーのホスト名}
# Username / Password を入力

docker buildx build --platform linux/amd64 \
  -t {コンテナレジストリー}/{イメージ名}:{バージョン} --push .

# 例
docker buildx build --platform linux/amd64 \
  -t dok-tutorial.sakuracr.jp/ollama-dynamic:latest --push .
```

push 後、コントロールパネルの「レジストリー認証情報」にホスト名 / ユーザー名 / パスワードを登録し、
タスク作成時に `registry` フィールドでその UUID を指定する。

（出典: https://manual.sakura.ad.jp/cloud/koukaryoku-container/tutorial/with-storage-and-registry.html ,
https://manual.sakura.ad.jp/cloud/koukaryoku-container/using-private-registries.html ）

### 入出力データの定石

学習データはオブジェクトストレージから取得し、成果物は `$SAKURA_ARTIFACT_DIR` に書く。
公式チュートリアルは、コンテナに AWS CLI を同梱して S3 互換 API で取得する構成を示している。

```dockerfile
FROM ollama/ollama:latest
ENTRYPOINT ["/entrypoint.sh"]

# AWS CLI を追加
RUN apt-get update && apt-get install -y unzip curl && \
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.22.35.zip" -o "awscliv2.zip" && \
    unzip awscliv2.zip && ./aws/install && \
    rm -rf awscliv2.zip aws/ /var/lib/apt/lists/*

WORKDIR /model
ENV S3_BUCKET=dok-tutorial
ENV AWS_DEFAULT_REGION=jp-north-1
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
```

```bash
#!/bin/bash
set -e
AWS_OPTS=""
[ -n "$AWS_ENDPOINT_URL" ] && AWS_OPTS="--endpoint-url ${AWS_ENDPOINT_URL}"
aws $AWS_OPTS s3 cp s3://${S3_BUCKET}/${MODEL_KEY} /model/model.gguf
# ... 学習・推論 ...
# 成果物は $SAKURA_ARTIFACT_DIR に書く
```

タスク作成時に渡す環境変数（推奨パターン）:

| 環境変数 | 値の例 |
|---|---|
| `AWS_ACCESS_KEY_ID` | オブジェクトストレージのアクセスキー |
| `AWS_SECRET_ACCESS_KEY` | 同シークレット |
| `AWS_DEFAULT_REGION` | `jp-north-1` |
| `AWS_ENDPOINT_URL` | `https://s3.isk01.sakurastorage.jp` |
| `S3_BUCKET` | バケット名 |

### 利用開始手順

さくらインターネット会員 ID とクラウドプロジェクトを準備 → サービストップページへログインして
プロジェクトを選択 → サービス約款に同意（初回のみ）。
（出典: https://manual.sakura.ad.jp/cloud/koukaryoku-container/getting-started.html ）

### 仕様・制限

出典: https://manual.sakura.ad.jp/cloud/koukaryoku-container/specification.html , https://manual.sakura.ad.jp/cloud/koukaryoku-container/faqs.html

| 項目 | 値 |
|---|---|
| コンテナイメージサイズ上限 | 30GB |
| 1 タスクの GPU 数上限 | 1（`h100-8gpu-80gb` は例外で 8） |
| アーティファクトのダウンロード期限 | 72 時間 |
| タスク最大実行時間 | 無制限（デフォルト 20 日で強制終了） |
| 同時実行タスク | 1 プロジェクトにつき 1 タスク（超過分は待機） |
| プラットフォーム | `linux/amd64` |

**非対応**（公式に「ウェブサービス作成のような汎用コンテナとしては設計されていません」と明記）:

- 異常終了時の自動再起動
- 負荷分散
- 永続稼働
- タスク間通信

### 課金

```
タスクの実行料金 = プランごとの単価 × コンテナの実行時間[秒]（最低利用時間あり）
1ヶ月の料金 = 当月内に実行したタスクの日別累積の合計
```

単価はマニュアルではなく製品サイトを参照: https://www.sakura.ad.jp/koukaryoku-dok/
（出典: https://manual.sakura.ad.jp/cloud/koukaryoku-container/pricing.html ）

### 外部接続（リッスンポート）

- 外部からのアクセスは **HTTPS のみ**（インターネット〜さくら設備間が HTTPS 443/TCP、さくら設備〜コンテナ間が HTTP）。
- Jupyter Notebook の一時起動用途を想定した機能。常時公開向けではない。使用後はタスクを中断すること。
  （出典: https://manual.sakura.ad.jp/cloud/koukaryoku-container/listening-port.html ）

---

## ストレージ

### オブジェクトストレージ（S3 互換）

出典: https://manual.sakura.ad.jp/cloud/objectstorage/about.html , https://manual.sakura.ad.jp/cloud/objectstorage/api.html

- Amazon S3 プロトコル採用（署名バージョン `s3v4`）。
- エンドポイント:
  - 石狩第1サイト: `s3.isk01.sakurastorage.jp`
  - 東京第1サイト: `s3.tky01.sakurastorage.jp`
- API は 2 種類:
  - **さくらのオブジェクトストレージ API**: バケット作成など。APIキー認証。
  - **Amazon S3 互換 API**: 作成済みバケット / オブジェクトの操作。バケット作成時に発行されるアクセスキーを使う。
- 課金はバケット単位・月額。該当請求月内で利用した**最大時の容量**が適用される。

制限:

| 項目 | 上限 |
|---|---|
| 1 プロジェクトあたりのバケット数 | 20 |
| 1 バケットの容量 | 10TiB |
| 1 バケットのオブジェクト数 | 1,000 万 |
| 1 オブジェクトのサイズ | 5TiB |

（サポート相談で緩和可能）

### バケットとアクセスキーの作成

- バケット: コントロールパネル左メニューの「バケット」→「バケットの追加」。バケット名は半角 3〜63 文字、
  英小文字 / 数字 / ダッシュ。
- アクセスキー: 「パーミッション」メニューから、バケットごとに権限を指定して発行する。
  権限は READ / WRITE / READ・WRITE / NONE（デフォルト）の 4 種。
  **アクセスキーの発行は 1 パーミッションあたり 1 個のみ。**

（出典: https://manual.sakura.ad.jp/cloud/objectstorage/basic.html ）

### リージョン名

| リージョン | 値 |
|---|---|
| 石狩 | `jp-north-1` |
| 東京 | `jp-east-1` |

### AWS CLI での操作

```bash
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="jp-north-1"
EP="https://s3.isk01.sakurastorage.jp"

aws --endpoint-url="$EP" s3 ls
aws --endpoint-url="$EP" s3 cp ./dataset.tar s3://my-bucket/
aws --endpoint-url="$EP" s3 ls s3://my-bucket/ --summarize --recursive
aws --endpoint-url="$EP" s3 rm s3://my-bucket/old.ckpt
```

未完了のマルチパートアップロードは容量として残り課金対象になるので、定期的に確認・中断する:

```bash
aws s3api list-multipart-uploads --bucket 対象バケット名 \
  --endpoint-url=https://s3.isk01.sakurastorage.jp --output json

aws s3api abort-multipart-upload --bucket 対象バケット名 \
  --upload-id "****" --key "file.dat" \
  --endpoint-url=https://s3.isk01.sakurastorage.jp
```

（出典: https://manual.sakura.ad.jp/cloud/objectstorage/faq.html ）

> s3cmd / rclone の設定例は公式マニュアルには見当たらない。上記のエンドポイント・リージョン名・
> 署名バージョン `s3v4` をそのまま設定すれば使える。

### NFS アプライアンス

出典: https://manual.sakura.ad.jp/cloud/appliance/nfs/index.html

- さくらのクラウドのスイッチ配下のローカルネットワーク内に設置する NFS ファイルサーバ。セットアップ済みで提供される。
- 標準プラン（HDD）: 100GB〜12TB（石狩第1・石狩第3・東京第2は 2TB〜12TB のみ）
- SSD プラン: 20GB〜4TB
- 複数サーバからの共用ストレージとして使う。

### 使い分け

| 置きたいもの | 置き場所 |
|---|---|
| OS / 環境そのもの | ディスク（ブロックストレージ） |
| 学習データ・チェックポイント・成果物 | オブジェクトストレージ |
| 複数 GPU サーバで共有したい作業領域 | NFS アプライアンス |
| DOK タスクの出力 | `SAKURA_ARTIFACT_DIR`（72 時間以内に回収） |
| VM の高速スクラッチ | 付属 NVMe（**揮発するので捨ててよいものだけ**） |

---

## ネットワーク

出典: https://manual.sakura.ad.jp/cloud/network/switch/about.html , https://manual.sakura.ad.jp/cloud/network/packet-filter.html

- **スイッチ**: LAN 内で使用する L2 スイッチ。プライベートネットワーク用。
- **ルータ+スイッチ**: インターネット回線とグローバル IP がバンドルされた L3 スイッチ。スタティックルート、IPv6 割り当てなどの拡張機能あり。
- グローバル IP は**月額料金のみ**（日割・時間割なし）。
- 追加 IP のうち、ネットワークアドレス 1 個・ブロードキャストアドレス 1 個・ゲートウェイアドレス 3 個の計 5 個は予約されて利用できない。

### パケットフィルタ

- サーバの仮想 NIC に着信するパケットを条件でフィルタする機能。ルールは**上から順に評価**、Allow / Deny。
- 1 パケットフィルタあたり**最大 30 ルール**。
- 手順: (1) パケットフィルタ作成 → (2) ルール設定 → (3) NIC へ適用
- SSH を絞るときは `SSH_Source_Network` プリセットに許可する CIDR を入れる。
- DHCP を使う場合は 67/UDP・68/UDP の疎通が必要。誤って塞ぐとネットワークが機能しなくなる。

---

## 課金の考え方

出典: https://manual.sakura.ad.jp/cloud/payment/server-charge.html , https://cloud.sakura.ad.jp/payment/

- 換算ルール: **時間割 10 時間 = 日割 1 日分**、**日割 20 日分 = 月額 1 か月分**。
- 月間利用時間に対して「時間割 + 日割の合算額」と「月額料金」を比較し、**安い方が採用される**。
- サーバは**起動時に課金開始・停止時に課金終了**。正確な時刻はイベントログの「状態遷移」を参照。
- ディスクは**作成完了から削除まで**課金される（サーバを停止してもディスク代は止まらない）。
- データ転送量による従量課金はなし。

金額を確認する場所:

- 料金 https://cloud.sakura.ad.jp/payment/
- 料金シミュレーション https://cloud.sakura.ad.jp/payment/simulation/
- 高火力DOK https://www.sakura.ad.jp/koukaryoku-dok/
- 高火力PHY https://www.sakura.ad.jp/koukaryoku-phy/

---

## 高火力PHY（参考）

出典: https://manual.sakura.ad.jp/ds/phy/guide_book/rk-phy/index.html

- 「生成系 AI の学習などの GPU リソースを提供する専用サーバーのサービス」。ベアメタル。
- 申し込みから最短 10 分、**最低利用期間 2 か月〜**。
- 確認できた GPU 搭載モデル: H100 8GPU（SYS-821GE-TNHR）、H200 141GB × 8、B200 180GB × 8。
- 「さくらの専用サーバ PHY」と共通の基盤・コントロールパネルで稼働する。
- 対応 OS はモデルごとに異なる。https://server.sakura.ad.jp/specification/os/ を参照。
- 申し込み: サービスサイト https://ai.sakura.ad.jp/gpu/koukaryoku-phy/ の「お申し込み」から
  https://secure.sakura.ad.jp/dedicated/phy/order/home へ。
  流れは「新規会員登録 → プロジェクト作成（要クレジットカード登録）→ お支払い（請求書発行・入金確認後に
  利用案内送付）→ 利用開始」。
- API: 「さくらの専用サーバ PHY」の API をそのまま使う。サーバ一覧取得、電源 On/Off のスケジュール実行、
  OS インストールやネットワーク設定を含むセットアップ自動化ができる。専用サーバ PHY 利用中であれば
  申し込み不要・無料。 https://manual.sakura.ad.jp/ds/phy/api/index.html

このスキルでは PHY の運用手順までは扱わない。申し込み・仕様確認は上記マニュアルと製品サイトを直接見ること。

---

## 参照リンク

- さくらのクラウド マニュアル https://manual.sakura.ad.jp/cloud/
- さくらのクラウド API (1.1) https://manual.sakura.ad.jp/cloud-api/1.1/
- usacloud ドキュメント https://docs.usacloud.jp/usacloud/
- 高火力DOK マニュアル https://manual.sakura.ad.jp/cloud/manual-koukaryoku-container.html
- 高火力DOK API 仕様 https://manual.sakura.ad.jp/koukaryoku-dok-api/spec.html
- 高火力VRT / GPUプラン https://manual.sakura.ad.jp/cloud/server/gpu-plan.html
- 高火力PHY マニュアル https://manual.sakura.ad.jp/ds/phy/
- オブジェクトストレージ https://manual.sakura.ad.jp/cloud/objectstorage/
- Terraform Provider https://registry.terraform.io/providers/sacloud/sakuracloud/latest/docs
