---
name: abci
description: ABCI 3.0（産業技術総合研究所のAI橋渡しクラウド）の操作・ジョブ投入を行うときに使う。ログイン(as.v3.abci.ai 経由の多段SSH)、バッチジョブ(qsub/qstat/qdel)、インタラクティブジョブ(qsub -I)、ジョブスクリプト作成、資源タイプ(rt_HF/rt_HG/rt_HC)、サービス種別(Spot/On-demand/Reserved)、moduleコマンド、ストレージ(/home /groups $PBS_LOCALDIR /beeond)、H200 GPU、MPI/NCCL、ABCIポイント(show_point)。「ABCI」「産総研のスパコン」「AI橋渡しクラウド」「rt_HF」「PBS」「#PBS」等が出てきたら参照。詳細は references/handbook.md。
---

# ABCI 3.0

産業技術総合研究所（AIST）の AI 向け計算基盤「AI橋渡しクラウド（ABCI）」の第3世代。
計算ノード 766 台、各ノードに **NVIDIA H200 SXM 141GB × 8**。

ジョブスケジューラ: **Altair PBS Professional**
- ディレクティブは `#PBS`（`#$` は UGE の記法で **ABCI 3.0 では機能しない**）
- グループ指定は `-P`（ABCI 2.0 / TSUBAME の `-g` ではない）
- Slurm ではないので `sbatch` / `squeue` は使えない

詳細リファレンス → `references/handbook.md`
公式ドキュメント → https://docs.abci.ai/v3/ja/
利用者ポータル → https://portal.v3.abci.ai/

## ログイン

アクセスサーバ `as.v3.abci.ai` を経由する**多段 SSH**（インタラクティブノードへの直接接続は不可）。

```bash
# 方法1: トンネルを張ってから接続
ssh -i /path/identity_file -L 50022:login:22 -l username as.v3.abci.ai
# 別ターミナルで
ssh -i /path/identity_file -p 50022 -l username localhost
```

~/.ssh/config 設定例（ProxyJump、OpenSSH 7.3 以降）:

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

設定後は `ssh abci` / `scp local-file abci:remote-dir` だけで済む。

- 認証は SSH 公開鍵のみ。鍵は利用者ポータルで登録（RSA 2048bit 以上 / ECDSA / Ed25519）
- ログインシェルは bash 固定
- インタラクティブノードは GPU なし・全利用者で共有。**高負荷処理は強制終了される**

## ジョブ投入

```bash
qsub -P <グループ> -q rt_HF -l select=1 job.sh    # バッチジョブ（Spot）
qstat                                              # 自分のジョブ一覧
qstat -f <ジョブID>                                 # 詳細（エラー原因確認）
qdel <ジョブID>                                     # 削除
qgstat                                             # グループ全体のジョブ一覧
nodestatus                                         # 空きノード数を確認
show_point                                         # ABCIポイント残量
```

`-P`（グループ）・`-q`（資源タイプ）・`-l select`（ノード数）は**すべて指定必須**。

インタラクティブジョブ（On-demand サービス）:

```bash
qsub -I -P <グループ> -q rt_HF -l select=1 -l walltime=1:00:00
qsub -IX -P <グループ> -q rt_HG -l select=1        # X転送あり
```

ジョブ状態: `R`=実行中 / `Q`=待機中 / `F`=完了 / `S`=一時停止 / `E`=終了処理中

## ジョブスクリプト雛形

```bash
#!/bin/sh
#PBS -q rt_HF             # 資源タイプ（必須）
#PBS -l select=1          # ノード数（必須）
#PBS -l walltime=1:23:45  # 経過時間制限（既定 1:00:00）
#PBS -P grpname           # ABCI利用グループ（必須）
#PBS -N myjob             # ジョブ名
#PBS -j oe                # 標準エラーを標準出力にマージ

cd ${PBS_O_WORKDIR}

source /etc/profile.d/modules.sh   # module を使う前に必須
module load cuda/12.6/12.6.1
./a.out
```

- `-P` はスクリプト内の `#PBS` でも `qsub` の引数でもよい
- `module` は `source /etc/profile.d/modules.sh` を**先に実行しないと使えない**

## 資源タイプ

`-q` に指定するキュー名がそのまま資源タイプ名。

| 資源タイプ | 論理CPUコア | メモリ | GPU | ローカルSSD | 標準利用の単価(Spot/On-demand) |
|---|---|---|---|---|---|
| `rt_HF` | 192 | 1920GB | H200×8 | 14TB | 16 pt/時間 |
| `rt_HG` | 16 | 160GB | H200×1 | 1.4TB | 3 pt/時間 |
| `rt_HC` | 32 | 320GB | なし | 1.4TB | 1 pt/時間 |

料金には「標準利用」と「開発加速利用」（上表の約半額）の2階層がある → `references/handbook.md`

`-l select` の書式: `select=<ノード数>[:ncpus=<コア数>:mpiprocs=<MPIプロセス数>:ompthreads=<スレッド数>]`

```bash
-l select=1                                       # rt_HF 1ノード占有
-l select=4:mpiprocs=48                           # rt_HF 4ノード、1ノード48プロセス
-l select=1:ncpus=16:mpiprocs=1:ompthreads=16     # rt_HG
-l select=1:ncpus=32:mpiprocs=1:ompthreads=32     # rt_HC
```

`ncpus` は資源タイプで決まる値が自動設定され、利用者が増やすことはできない。
`rt_HG` / `rt_HC` はノード共有のため **1ノードのみ**（複数ノード確保は `rt_HF`）。

## サービス種別

| サービス | 用途 | 指定方法 | 経過時間上限 |
|---|---|---|---|
| **Spot** | 通常のバッチジョブ | `qsub -P g -q rt_HF -l select=1 job.sh` | 168時間 |
| **On-demand** | 対話利用・デバッグ | `qsub -I ...`（`-I` を付ける） | 12時間 |
| **Reserved** | 日単位の事前予約 | `qsub -q R1234 -v RTYPE=rt_HF ...` | 予約終了まで |

```bash
qsub -p 20 ...                                  # Spot 優先実行（課金係数 1.5倍）
qrsub -R 260401 -D 7 -P grpname -n 4 -N "name"  # 4ノードを7日間予約
qrstat                                          # 予約一覧（予約IDを確認）
qrdel R1234.pbs1                                # 予約取消
```

予約ノードへの投入は `-q <予約ID>` と `-v RTYPE=<資源タイプ>` の**両方が必須**。

## モジュール

```bash
source /etc/profile.d/modules.sh    # 先に必須（スクリプト内でも）
module avail                        # 利用可能一覧
module list                         # ロード中を確認
module purge                        # 全クリア
module load cuda/12.6/12.6.1        # CUDA（バージョンは 系列/詳細 の2段表記）
module load cudnn/9.5/9.5.1
module load nccl/2.23/2.23.4-1
module load hpcx/2.20               # NVIDIA HPC-X (Open MPI ベース)
module load intel-mpi/2021.13
module load gcc/13.2.0              # 他に intel/2024.2.1, nvhpc/24.9
module load python/3.12/3.12.9
module load singularity-ce/4.3.6    # コンテナ
```

バージョン表記は `cuda/12.6/12.6.1` のように **系列/詳細** の2段。実際に入っている版は `module avail` で確認する。

## ストレージ

| パス | 種別 | 容量目安 | 注意 |
|---|---|---|---|
| `$HOME` (`/home/<user>`) | Lustre | 2TiB | 全利用者共有 |
| `/groups/<グループ>` | Lustre | 申請制 | グループ共有・大容量 |
| `$PBS_LOCALDIR` | ノードローカル NVMe | rt_HF:14TB / 他:1.4TB | **ジョブ終了時に自動削除** |
| `/beeond` | BeeOND（複数ノード共有） | 割当ノード分を集約 | `-v BEEOND_ON=1` + rt_HF が必要。**ジョブ終了時に削除** |
| `/groups_s3/<グループ>` | クラウドストレージ(S3互換) | 申請制 | `show_cs_quota` で確認 |

```bash
show_quota           # ホーム・グループ領域の使用量（-b G で GiB 表示）
show_cs_quota        # クラウドストレージ使用量
show_point           # ABCIポイント残量
show_point_history -g <グループ>   # 月次履歴
```

- `$PBS_LOCALDIR` の中身は消えるので、必要なファイルはジョブ内で `$HOME` / `/groups` へ `cp` する
- Lustre 上に小ファイルを大量作成しない（HDF5 / WebDataset 等にまとめる）

## 利用制限

| 項目 | 値 |
|---|---|
| 同時投入可能ジョブ数 / ユーザ | 1,000 |
| 同時実行ジョブ数 / ユーザ | 200 |
| 1アレイジョブの最大タスク数 | 75,000 |
| 同時利用ノード数（rt_HF） | 1〜128 |
| 同時利用ノード数（rt_HG / rt_HC） | 1 |
| 最大経過時間 | Spot 168h / On-demand 12h |
| 予約 | 1〜60日、グループあたり最大32ノード |

制限値は更新されることがある。最新は https://docs.abci.ai/v3/ja/job-execution/ を確認。

## よくあるエラー

| 症状 | 対処 |
|---|---|
| `qsub: ... -P/-q/-l select` 関連のエラー | 3つとも必須。いずれか欠けていないか確認 |
| ジョブが `Q` のまま進まない | `nodestatus` で空き確認、`qstat -f <ID>` で理由確認 |
| ジョブ実行に失敗する | `show_point` でポイント残量を確認（不足だと実行に失敗する） |
| `Disk quota exceeded` | `show_quota` で使用量・inode を確認して不要ファイル削除 |
| `#$ -l rt_F=1` が効かない | ABCI 2.0 の記法。3.0 は `#PBS -q rt_HF -l select=1` |
| `-g grpname` が通らない | 3.0 のグループ指定は `-P grpname` |
| `module: command not found` | `source /etc/profile.d/modules.sh` を先に実行 |
| `cannot be loaded due to missing prereq` | 依存モジュールを先に load（`module show <mod>` で確認） |
| rt_HF のノードに ssh できない | `-v USE_SSH=1`（自分のみ）または `2`（グループメンバー）を付けて投入 |

## ABCI 2.0 との違い（混同注意）

| 項目 | ABCI 2.0（旧） | ABCI 3.0（現行） |
|---|---|---|
| スケジューラ | Grid Engine 系 | PBS Professional |
| ディレクティブ | `#$` | `#PBS` |
| グループ指定 | `-g group` | `-P group` |
| 資源指定 | `-l rt_F=1`, `-l h_rt=1:23:45` | `-q rt_HF -l select=1`, `-l walltime=1:23:45` |
| 資源タイプ名 | `rt_F` / `rt_G.large` / `rt_C.small` 等 | `rt_HF` / `rt_HG` / `rt_HC` の3種のみ |
| インタラクティブ | `qrsh` | `qsub -I` |
| GPU | V100 / A100 | H200 SXM 141GB |

Web 上の ABCI 情報には 2.0 時代の記事が多く残っている。`rt_F` や `#$` が出てくる記事は 2.0 向けと判断する。
