# ABCI 3.0 詳細リファレンス

産業技術総合研究所（AIST）「AI橋渡しクラウド（ABCI）」第3世代。
公式ドキュメント: https://docs.abci.ai/v3/ja/

本書は公式ドキュメントの要点をまとめたもの。数値・制限値は改定されることがあるため、
重要な判断の前には公式ページで裏を取ること。

---

## システム概要

| 項目 | 内容 |
|---|---|
| 計算ノード | HPE Cray XD670 × 766台（hnode001〜hnode766） |
| CPU | Intel Xeon Platinum 8558 (2.1GHz, 48コア) × 2 |
| メモリ | 64GB DDR5-5600 × 32枚（計 2TB） |
| GPU | NVIDIA H200 SXM 141GB × 8（全システムで 6,128基） |
| ローカルSSD | NVMe 7.68TB × 2（`/local` として 14TB、XFS） |
| インターコネクト | InfiniBand NDR (200Gbps) × 8 + HDR × 1 |
| インタラクティブノード | HPE ProLiant DL380 Gen11 × 5（Xeon Platinum 8468 ×2、GPUなし） |
| ストレージ | DDN ES400NVX2、実効約75PB（/home 10PB, /groups 63PB, /groups_s3 1PB） |

出典: https://docs.abci.ai/v3/ja/system-overview/

計算ノード(H) の GPU は 500W に電力キャッピングを施した状態で提供される
（出典: https://abci.ai/ja/how_to_use/tariffs.html ）。

---

## ログイン・初期設定

### 接続経路

```
手元のPC --(SSH)--> アクセスサーバ as.v3.abci.ai --(SSH)--> インタラクティブノード login
```

二段階の公開鍵認証。インタラクティブノードへの直接接続はできない。

### 方法1: ポートフォワード

```bash
# ターミナル1: トンネルを張る（接続したままにする）
ssh -i /path/identity_file -L 50022:login:22 -l username as.v3.abci.ai

# ターミナル2: ローカルポート経由でログイン
ssh -i /path/identity_file -p 50022 -l username localhost
```

ファイル転送も同じポートを使う:

```bash
scp -i /path/identity_file -P 50022 local-file username@localhost:remote-dir
```

### 方法2: ProxyJump（推奨・OpenSSH 7.3 以降）

`~/.ssh/config`:

```
Host abci
    HostName login
    User username
    ProxyJump %r@as.v3.abci.ai
    IdentityFile /path/to/identity_file
    HostKeyAlgorithms ssh-ed25519

Host as.v3.abci.ai
    IdentityFile /path/to/identity_file
```

```bash
ssh abci
scp local-file abci:remote-dir
rsync -av ./data/ abci:/groups/grpname/data/
```

Windows の OpenSSH_for_Windows 7.7p1 は `ProxyJump` に対応しないため `ProxyCommand` を使う。

### SSH 鍵

- 利用者ポータル https://portal.v3.abci.ai/ で公開鍵を登録
- 対応: RSA（2048bit 以上） / ECDSA（256, 384, 521bit） / Ed25519
- ログインシェルは bash 固定（変更したい場合は abci3-qa@abci.ai へ連絡）

### インタラクティブノードでの注意

- CPU・メモリは全利用者で共有。高負荷処理を行うとシステムにより**強制終了される**
  （出典: https://docs.abci.ai/v3/ja/system-overview/ ）
- GPU は搭載されていない
- コンパイル・デバッグ・可視化などの重い処理は On-demand サービス（`qsub -I`）で計算ノードを使う

---

## ジョブスケジューラ詳細

Altair PBS Professional。

### コマンド一覧

| コマンド | 用途 |
|---|---|
| `qsub [options] [script]` | ジョブ投入 |
| `qstat [-f｜-a｜-x｜-t｜-q]` | ジョブ状態確認（`-x` は終了済みも表示） |
| `qstat -f <ジョブID>` | ジョブ詳細（待機理由・エラー確認） |
| `qdel <ジョブID>` | ジョブ削除 |
| `qgstat [options]` | グループ内の全ジョブを確認 |
| `qgstat_l [-f] [job-id]` | `qgstat` の軽量版（5分毎更新のキャッシュ） |
| `qgdel <ジョブID>` | グループ管理者によるジョブ削除 |
| `qrsub -R <YYMMDD> -D <日数> -P <グループ> -n <ノード数> -N <名前>` | ノード予約 |
| `qrstat [-f｜-F｜--available=<グループ>]` | 予約状況確認 |
| `qrdel <予約ID>` | 予約取消 |
| `nodestatus` | 利用可能ノード数の表示 |

### qsub オプション詳細

| オプション | 説明 |
|---|---|
| `-P group` | ABCI利用グループ。**必須** |
| `-q resource_type` | 資源タイプ（= キュー名）。**必須** |
| `-l select=num[:ncpus=..:mpiprocs=..:ompthreads=..]` | ノード数・MPIプロセス数・スレッド数。**必須** |
| `-l walltime=[HH:MM:]SS` | 経過時間制限（既定 1:00:00） |
| `-N name` | ジョブ名 |
| `-o stdout` / `-e stderr` | 標準出力・標準エラーのファイル名 |
| `-j oe` | 標準エラーを標準出力にマージ |
| `-k oe` | 実行中に標準出力・標準エラーをストリーム出力 |
| `-p priority` | Spot サービスの優先度（0 既定 / 20 優先・課金係数1.5） |
| `-J start-stop[:step]` | アレイジョブのインデックス範囲 |
| `-I` | インタラクティブジョブ |
| `-IX` | インタラクティブジョブ + X転送 |
| `-v RTYPE=resource_type` | 予約ジョブでの資源タイプ指定（予約投入時 **必須**） |
| `-v USE_SSH={0,1,2}` | rt_HF で計算ノードへの SSH を許可（0=不可 / 1=自分 / 2=グループメンバー） |
| `-v BEEOND_ON=1` | BeeOND（共有スクラッチ）を有効化（rt_HF のみ） |
| `-M addr` / `-m {n,a,b,e}` | メール通知先 / 通知契機（a=中止時・既定, b=開始時, e=終了時） |

`-v` は `-v BEEOND_ON=1,USE_SSH=1` のようにカンマ区切りで併記できる。

### ジョブ状態

| 記号 | 状態 |
|---|---|
| `R` | 実行中 |
| `Q` | 待機中 |
| `F` | 完了 |
| `S` | 一時停止 |
| `E` | 終了処理中 |

### 環境変数

| 環境変数 | 内容 |
|---|---|
| `PBS_ENVIRONMENT` | バッチなら `PBS_BATCH`、インタラクティブなら `PBS_INTERACTIVE` |
| `PBS_JOBID` | ジョブID |
| `PBS_JOBNAME` | ジョブ名 |
| `PBS_NODEFILE` | 割当ホスト一覧ファイルへのパス（MPI の `-hostfile` に渡す） |
| `PBS_LOCALDIR` | 割当ローカルストレージへのパス |
| `PBS_O_WORKDIR` | ジョブ投入時の作業ディレクトリ |
| `PBS_ARRAY_INDEX` | アレイジョブのインデックス |

ジョブは投入ディレクトリでは実行されないため、スクリプト冒頭で `cd ${PBS_O_WORKDIR}` する。

---

## サービス種別

| サービス | 位置づけ | 使い方 | 経過時間 |
|---|---|---|---|
| Spot | 対話不要なバッチジョブ | `qsub -P g -q rt_HF -l select=1 job.sh` | 上限 168:00:00 / 既定 1:00:00 |
| On-demand | コンパイル・デバッグ・対話利用・可視化 | `qsub -I -P g -q rt_HF -l select=1` | 上限 12:00:00 / 既定 1:00:00 |
| Reserved | 日単位の事前予約。混雑の影響を受けない | `qrsub` で予約 → `qsub -q <予約ID> -v RTYPE=<資源タイプ>` | 予約終了時刻まで |

ノード時間積の上限: Spot 21,504 ノード時間 / On-demand 12 ノード時間。

### 予約（Reserved）の流れ

```bash
# 1. 予約する（2026-04-01 から 7日間、4ノード）
qrsub -R 260401 -D 7 -P grpname -n 4 -N "Reserve_for_AI"

# 2. 予約IDを確認（R1234 など）
qrstat
qrstat --available=grpname     # 予約可能な空き状況

# 3. 予約枠へ投入（-q に予約ID、-v RTYPE に資源タイプ）
qsub -P grpname -q R1234 -v RTYPE=rt_HF -l select=1 job.sh
qsub -I -P grpname -q R1234 -v RTYPE=rt_HG -l select=1

# 4. 取消
qrdel R1234.pbs1
```

予約設定: 最小1日・最大60日、1予約あたり 1〜32ノード、最大 5,376 ノード時間積。
グループあたり同時最大32ノード、システム全体で同時最大64ノード。

---

## ジョブスクリプトテンプレート集

公式ドキュメントに完全な例があるものは原文どおり、無いものは公式のオプション仕様から
組み立てた雛形（その旨を注記）を載せる。

### 基本形（公式）

```sh
#!/bin/sh
#PBS -q rt_HF
#PBS -l select=1
#PBS -l walltime=1:23:45
#PBS -P grpname

cd ${PBS_O_WORKDIR}

source /etc/profile.d/modules.sh
module load cuda/12.6/12.6.1
./a.out
```

### 実行中の出力をファイルに流す（公式）

```sh
#!/bin/sh
#PBS -q rt_HF
#PBS -l select=1
#PBS -l walltime=1:23:45
#PBS -P grpname

cd ${PBS_O_WORKDIR}

./a.out >& logfilename
```

`#PBS -k oe` を付けると PBS 側でストリーム出力される。

### GPU 1枚（rt_HG）※雛形

```sh
#!/bin/sh
#PBS -q rt_HG
#PBS -l select=1:ncpus=16:mpiprocs=1:ompthreads=16
#PBS -l walltime=2:00:00
#PBS -P grpname
#PBS -N single_gpu
#PBS -j oe

cd ${PBS_O_WORKDIR}

source /etc/profile.d/modules.sh
module load cuda/12.6/12.6.1
module load python/3.12/3.12.9

python train.py
```

### CPU のみ（rt_HC）※雛形

```sh
#!/bin/sh
#PBS -q rt_HC
#PBS -l select=1:ncpus=32:mpiprocs=1:ompthreads=32
#PBS -l walltime=1:00:00
#PBS -P grpname

cd ${PBS_O_WORKDIR}

export OMP_NUM_THREADS=32
./a.out
```

### MPI（複数ノード）※雛形

公式ドキュメントにはインタラクティブでの実行例が示されている:

```bash
qsub -I -P groupname -q rt_HF -l select=2:mpiprocs=192 -l walltime=01:00:00
module load hpcx/2.20
mpirun -np 2 -map-by ppr:1:node -hostfile $PBS_NODEFILE ./hello_c
```

バッチジョブにすると:

```sh
#!/bin/sh
#PBS -q rt_HF
#PBS -l select=4:mpiprocs=48
#PBS -l walltime=1:00:00
#PBS -P grpname
#PBS -N mpi_job
#PBS -j oe

cd ${PBS_O_WORKDIR}

source /etc/profile.d/modules.sh
module load hpcx/2.20

mpirun -np 192 -hostfile ${PBS_NODEFILE} ./a.out
```

`mpirun` / `mpiexec` には `-hostfile ${PBS_NODEFILE}` を必ず渡す。

### アレイジョブ ※雛形

```sh
#!/bin/sh
#PBS -q rt_HC
#PBS -l select=1:ncpus=32
#PBS -l walltime=1:00:00
#PBS -P grpname
#PBS -J 1-10

cd ${PBS_O_WORKDIR}
./a.out input_${PBS_ARRAY_INDEX}.dat
```

1アレイジョブあたり最大 75,000 タスク。

### ローカルスクラッチ活用（公式）

```bash
#!/bin/bash
#PBS -P grpname
#PBS -q rt_HF
#PBS -l select=1

echo test1 > $PBS_LOCALDIR/foo.txt
echo test2 > $PBS_LOCALDIR/bar.txt
cp -rp $PBS_LOCALDIR/foo.txt $HOME/test/foo.txt
```

`$PBS_LOCALDIR` 以下はジョブ終了時に削除される。必要なファイルはジョブ内で
ホーム領域またはグループ領域へコピーすること。

### BeeOND（複数ノードで共有するスクラッチ）（公式）

```bash
#!/bin/bash
#PBS -P grpname
#PBS -q rt_HF
#PBS -l select=2
#PBS -v BEEOND_ON=1

echo test1 > /beeond/foo.txt
echo test2 > /beeond/bar.txt
cp -rp /beeond/foo.txt $HOME/test/foo.txt
```

`beegfs copy` でまとめて配る例（公式）:

```bash
#!/bin/bash
#PBS -P grpname
#PBS -q rt_HF
#PBS -l select=2
#PBS -v BEEOND_ON=1,USE_SSH=1

cat $PBS_NODEFILE | sort -u > ./machinefile
beegfs copy -m ./machinefile ${HOME}/data /beeond/ -v 1
#(計算処理)
beegfs copy -m ./machinefile /beeond/data ${HOME}/ -v 1
```

BeeOND の利用には `-v BEEOND_ON=1` と `-q rt_HF` の両方が必要。`/beeond` もジョブ終了時に削除される。

---

## Environment Modules

```bash
source /etc/profile.d/modules.sh    # bash/sh。これを先に実行しないと module が使えない
source /etc/profile.d/modules.csh   # csh/tcsh
```

| サブコマンド | 用途 |
|---|---|
| `module avail` | 利用可能モジュール一覧 |
| `module list` | ロード済み一覧 |
| `module show <mod>` | 設定内容・依存関係を表示 |
| `module load <mod>` | ロード |
| `module unload <mod>` | アンロード |
| `module switch <A> <B>` | 入れ替え |
| `module purge` | 全アンロード |
| `module help <mod>` | 使い方 |

### 主要モジュール（2026年時点で公式ドキュメントに記載のあるもの）

| 分類 | モジュール名 | バージョン例 |
|---|---|---|
| コンパイラ | `gcc` | 13.2.0, 15.2.0 |
| コンパイラ | `intel` | 2024.2.1, 2025.3 |
| コンパイラ | `nvhpc` | 24.9, 26.3 |
| GPU | `cuda` | 11.8.0 〜 13.2/13.2.1 |
| GPU | `cudnn` | 9.5/9.5.1 〜 |
| GPU | `nccl` | 2.23/2.23.4-1 〜 |
| MPI | `hpcx` | 2.20, 2.26（派生: `hpcx-mt` / `hpcx-debug` / `hpcx-prof`） |
| MPI | `intel-mpi` | 2021.13, 2021.17 |
| 開発 | `cmake` | 4.1.1, 4.3.2 |
| 言語 | `python` | 3.12/3.12.9, 3.13/3.13.2, 3.14/3.14.4 |
| 言語 | `R` | 4.5.1, 4.5.3 |
| コンテナ | `singularity-ce` | 4.3.6, 4.4.1 |
| コンテナ | `singularitypro` | 4.1.12 |

バージョンは随時更新されるため、**実機で `module avail` を確認するのが確実**。

依存関係・排他関係は module 側で強制される:
- 前提モジュール未ロード → `cannot be loaded due to missing prereq.`
- 排他モジュールの同時ロード → `conflict` 警告

---

## MPI

利用可能な実装:
- **NVIDIA HPC-X**（`hpcx/2.20` は Open MPI 4.1.7a1 ベース。NCCL-SHARP プラグインは NCCL 2.23 相当）
- **Intel MPI**（`intel-mpi/2021.13` 等）

```bash
module load hpcx/2.20
mpirun -np 192 -hostfile ${PBS_NODEFILE} ./a.out
mpirun -np 2 -map-by ppr:1:node -hostfile ${PBS_NODEFILE} ./hello_c   # 1ノード1プロセス
```

### InfiniBand マルチレール制御

計算ノードは InfiniBand NDR × 8 を持つ。使用レーン数は環境変数で制御する。

| 環境変数 | 既定値 | 意味 |
|---|---|---|
| `UCX_MAX_RNDV_RAILS` | 4 | 大きいメッセージ（rendezvous）で使うレーン数（1〜8） |
| `UCX_MAX_EAGER_RAILS` | 1 | 小さいメッセージ（eager）で使うレーン数（1〜8） |
| `UCX_NET_DEVICES` | — | 使用デバイスの明示指定（`mlx5_ibn$i:1` 形式） |

出典: https://docs.abci.ai/v3/ja/mpi/

---

## ストレージ

| 領域 | パス | 容量 / Quota | 特性 |
|---|---|---|---|
| ホーム | `/home/<user>`（`$HOME`） | 2TiB | Lustre、永続 |
| グループ | `/groups/<グループ>` | 申請制（従量課金: 5 pt/TB・月） | Lustre、永続、グループ共有 |
| ローカルスクラッチ | `$PBS_LOCALDIR` | rt_HF 14TB / rt_HG・rt_HC 1.4TB | NVMe(XFS)、**ジョブ終了時に削除** |
| 共有スクラッチ | `/beeond` | 割当ノードのローカル容量を集約 | BeeGFS、**ジョブ終了時に削除** |
| クラウドストレージ | `/groups_s3/<グループ>` | 申請制 | S3互換オブジェクトストレージ |

```bash
show_quota          # ホーム・グループ領域（既定 TiB 表示。-b G で GiB）
show_cs_quota       # クラウドストレージ（-g GROUP -b unit -csv）
```

### Lustre 利用上の注意

- 小さいファイルを大量に作成・アクセスしない（HDF5、WebDataset 等にまとめる）
- 永続性の不要なデータはメモリ上で扱う
- 高速アクセスが必要なデータはスクラッチ領域へステージングする
- 同一ファイルの open/close の繰り返しを避ける
- 短期間に億を超えるファイルを作る場合は事前相談が必要

ストライプ設定:

```bash
lfs setstripe -S 1m -i 4 -c 4 stripe-file
lfs getstripe stripe-file
```

---

## コンテナ

SingularityCE / SingularityPRO が利用できる。Docker デーモンは使えないが、
Docker Hub のイメージは `docker://` で直接実行できる。

```bash
source /etc/profile.d/modules.sh
module load singularity-ce/4.3.6

singularity run --nv ./tensorflow.sif
singularity exec --nv ./tensorflow.sif python3 sample.py
singularity run --nv docker://tensorflow/tensorflow:latest-gpu   # Docker Hub から直接
```

- `--nv` で GPU を利用する
- コンテナ内から見える GPU は `SINGULARITYENV_CUDA_VISIBLE_DEVICES` で制御する

出典: https://docs.abci.ai/v3/ja/containers/

---

## 利用制限

| 項目 | 制限値 |
|---|---|
| 1ユーザあたりの同時投入可能ジョブ数 | 1,000 |
| 1ユーザあたりの同時実行ジョブ数 | 200 |
| 1アレイジョブあたりの最大タスク数 | 75,000 |
| システム同時実行数（rt_HF） | 736 |
| システム同時実行数（rt_HG） | 56 |
| システム同時実行数（rt_HC） | 38 |
| 同時利用ノード数（On-demand / Spot） | rt_HF: 1〜128、rt_HG / rt_HC: 1 |
| 経過時間（Spot） | 上限 168:00:00 / 既定 1:00:00 |
| 経過時間（On-demand） | 上限 12:00:00 / 既定 1:00:00 |
| ノード時間積（Spot） | 21,504 ノード時間 |
| ノード時間積（On-demand） | 12 ノード時間 |
| 予約（Reserved） | 1〜60日、1予約 1〜32ノード、グループ同時32ノード、システム同時64ノード |

※ システム同時実行数などは運用状況に応じて改定される。最新値は
https://docs.abci.ai/v3/ja/job-execution/ と https://docs.abci.ai/v3/ja/system-updates/ を参照。

---

## 課金（ABCIポイント）

料金には「標準利用」と「開発加速利用」の2階層がある（2026年度）。

| 資源タイプ | 標準利用 (Spot/On-demand) | 開発加速利用 (Spot/On-demand) | 標準利用 (Reserved) | 開発加速利用 (Reserved) |
|---|---|---|---|---|
| `rt_HF` | 16 pt/時間 | 7.5 pt/時間 | 576 pt/日 | 270 pt/日 |
| `rt_HG` | 3 pt/時間 | 1.5 pt/時間 | 設定なし | 設定なし |
| `rt_HC` | 1 pt/時間 | 0.5 pt/時間 | 設定なし | 設定なし |

ストレージ（グループ領域）: 標準利用 5 pt/TB・月 / 開発加速利用 2.5 pt/TB・月
ポイント単価: 220円（税込、2026年度）

- Spot の優先実行（`-p 20`）は課金係数 1.5 倍
- ポイント単価・料金体系は年度ごとに改定される → https://abci.ai/ja/how_to_use/tariffs.html
- ジョブ実行開始時に使用予定ポイントを減算し、終了時に実績値で再計算する
- ポイントが不足しているとジョブ実行に失敗する

```bash
show_point                          # 残量
show_point -g <グループ> -csv        # グループ単位・CSV出力
show_point_history -g <グループ>     # 月次履歴
```

---

## トラブルシュート

| 症状 | 確認・対処 |
|---|---|
| ジョブが `Q` から進まない | `qstat -f <ID>` で待機理由、`nodestatus` で空き状況 |
| ジョブが即失敗する | `show_point` でポイント残量。`-P` / `-q` / `-l select` の指定漏れ |
| `Disk quota exceeded` | `show_quota` で容量・inode の使用量を確認し、不要ファイルを削除 |
| `module: command not found` | `source /etc/profile.d/modules.sh` を先に実行 |
| `cannot be loaded due to missing prereq.` | 依存モジュールを先に load |
| `conflict` 警告 | 排他モジュール。`module purge` してから load し直す |
| 計算ノードに ssh できない | rt_HF で `-v USE_SSH=1`（自分）/ `2`（グループメンバー）を付けて投入 |
| BeeOND が使えない | `-q rt_HF` と `-v BEEOND_ON=1` の両方が必要 |
| 2.0 向けの記法が動かない | `#$` → `#PBS`、`-g` → `-P`、`rt_F` → `rt_HF`、`qrsh` → `qsub -I` |

既知の問題は https://docs.abci.ai/v3/ja/known-issues/ に掲載される（例: BeeOND の
`beegfs copy -m` でファイル権限が 777 になる問題など）。

---

## ABCI 2.0 からの移行

| 項目 | ABCI 2.0 | ABCI 3.0 |
|---|---|---|
| スケジューラ | Grid Engine 系 | Altair PBS Professional |
| ディレクティブ | `#$` | `#PBS` |
| グループ指定 | `-g group` | `-P group` |
| 資源タイプ指定 | `-l rt_F=1` | `-q rt_HF -l select=1` |
| 経過時間 | `-l h_rt=1:23:45` | `-l walltime=1:23:45` |
| 資源タイプ名 | `rt_F` / `rt_G.large` / `rt_G.small` / `rt_C.large` / `rt_C.small` / `rt_AF` / `rt_AG.small` | `rt_HF` / `rt_HG` / `rt_HC` |
| インタラクティブジョブ | `qrsh` | `qsub -I` |
| ローカルスクラッチ | `$SGE_LOCALDIR` | `$PBS_LOCALDIR` |
| GPU | V100 / A100 | H200 SXM 141GB |

Web 検索で出てくる ABCI の記事は 2.0 向けが多い。`rt_F` や `#$` が出てきたら 2.0 向けと判断する。

---

## 参照リンク

- ユーザーガイド（3.0）: https://docs.abci.ai/v3/ja/
- システム概要: https://docs.abci.ai/v3/ja/system-overview/
- 利用開始: https://docs.abci.ai/v3/ja/getting-started/
- ジョブ実行: https://docs.abci.ai/v3/ja/job-execution/
- ストレージ: https://docs.abci.ai/v3/ja/storage/
- Environment Modules: https://docs.abci.ai/v3/ja/environment-modules/
- MPI: https://docs.abci.ai/v3/ja/mpi/
- コンテナ: https://docs.abci.ai/v3/ja/containers/
- 既知の問題: https://docs.abci.ai/v3/ja/known-issues/
- システム更新履歴: https://docs.abci.ai/v3/ja/system-updates/
- 料金: https://abci.ai/ja/how_to_use/tariffs.html
- FAQ: https://abci.ai/ja/how_to_use/faq.html
- 利用者ポータル: https://portal.v3.abci.ai/
