---
name: wisteria
description: Wisteria/BDEC-01（東京大学情報基盤センターのスーパーコンピュータ）の操作・ジョブ投入を行うときに使う。ログイン(wisteria.cc.u-tokyo.ac.jp、公開鍵認証)、バッチジョブ(pjsub/pjstat/pjdel)、インタラクティブジョブ(pjsub --interact)、ジョブスクリプト(#PJM)作成、リソースグループ(regular-o/short-o/debug-o/regular-a/share等)、Odyssey(A64FX)とAquarius(A100)の違い、クロスコンパイル(frtpx/fccpx/mpifccpx)、moduleコマンド、ストレージ(/home /work)、トークン(show_token)、Singularity。「Wisteria」「BDEC」「Odyssey」「Aquarius」「東大のスパコン」「pjsub」「#PJM」「A64FX」「富士通TCS」等が出てきたら参照。詳細は references/handbook.md。
---

# Wisteria/BDEC-01

東京大学情報基盤センターのスーパーコンピュータ。**性格の違う2つのサブシステム**からなる。

| | Wisteria-O（**Odyssey**） | Wisteria-A（**Aquarius**） |
|---|---|---|
| 用途 | シミュレーション（CPU） | データ・学習・解析（GPU） |
| ノード | PRIMEHPC FX1000 × 7,680 | PRIMERGY GX2570 M6 × 45 |
| CPU | **A64FX**（Arm / aarch64）48コア 2.2GHz | Xeon Platinum 8360Y (Ice Lake) 36コア × 2 |
| メモリ | 32GiB | 512GiB |
| GPU | なし | **NVIDIA A100 40GB × 8** |
| ネットワーク | Tofu インターコネクト D | InfiniBand HDR 200Gbps × 4 |

ジョブスケジューラ: **富士通 Technical Computing Suite (TCS) / PJM**
- ディレクティブは `#PJM`。Slurm（`sbatch`）でも PBS/UGE（`qsub`）でもない
- グループ（課金先）は `-g <グループ名>` で**必ず指定**する
- Oakbridge-CX・Oakforest-PACS・Reedbush 用のスクリプトとは**互換性がない**

詳細リファレンス → `references/handbook.md`
公式サービスページ → https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/service/
利用支援ポータル（要ログイン、利用手引書はここにしかない） → https://wisteria-www.cc.u-tokyo.ac.jp/

## ログイン

```bash
ssh -l <ユーザ名> wisteria.cc.u-tokyo.ac.jp
```

~/.ssh/config 設定例:

```
Host wisteria
    HostName wisteria.cc.u-tokyo.ac.jp
    User <ユーザ名>
    IdentityFile ~/.ssh/id_ed25519
```

- 認証は **SSH 公開鍵のみ**。鍵は利用支援ポータルで登録する（ポータルの初回ログインは書面／メールで届く初期パスワード）
- ログインシェルは bash（`chsh` で変更可）
- ログインノードは **x86_64（Cascade Lake）**。Odyssey・Aquarius 共通の入口
- `~/.ssh` のパーミッションを壊すと自力で復旧できない（サポートへ連絡）。触る前にバックアップする

## Odyssey と Aquarius の最大の違い：クロスコンパイル

| | Odyssey | Aquarius |
|---|---|---|
| 計算ノードの命令セット | aarch64（A64FX, SVE） | x86_64（ログインノードとほぼ同じ） |
| ログインノードでのビルド | **クロスコンパイル必須** | 普通にビルドできる |
| 先に読むモジュール | `module load odyssey` | `module load aquarius` |

Odyssey 用の富士通コンパイラ（`module load odyssey` または `module load fj`。富士通 MPI も自動でロードされる）:

| 言語 | ログインノード（クロス） | 計算ノード上（ネイティブ） |
|---|---|---|
| C | `fccpx` | `fcc` |
| C++ | `FCCpx` | `FCC` |
| Fortran | `frtpx` | `frt` |
| MPI | `mpifccpx` / `mpiFCCpx` / `mpifrtpx` | `mpifcc` / `mpiFCC` / `mpifrt` |

```bash
module load odyssey
mpifccpx -Kfast -Koptmsg=2 -o hello hello.c   # ログインノードで aarch64 バイナリを作る
```

ログインノードで作った Odyssey 用バイナリは**ログインノードでは実行できない**（`cannot execute binary file`）。
逆に、ログインノード用にビルドしたもの（pip で入れた x86 のパッケージなど）は Odyssey では動かない。

## ジョブ投入

```bash
pjsub job.sh                    # バッチジョブ投入（-g は #PJM に書くか引数で渡す）
pjsub -g <グループ> job.sh
pjstat                          # 自分のジョブ一覧（使用トークンも表示）
pjdel <ジョブID>                 # 削除
pjstat -H                       # 過去のジョブ履歴
pjstat --rsc                    # 投入できるリソースグループ一覧
pjstat --rsc -x                 # リソースグループの詳細構成
pjstat --rscuse                 # 計算ノードの混み具合
pjstat --limit                  # グループの同時投入／実行上限
show_token                      # トークン残量
```

インタラクティブジョブ（トークンを消費しない）:

```bash
pjsub --interact -g <グループ> -L rscgrp=interactive-o,elapse=00:30:00          # Odyssey 1ノード
pjsub --interact -g <グループ> -L rscgrp=interactive-o,node=12,elapse=00:10:00  # Odyssey 最大12ノード
pjsub --interact -g <グループ> -L rscgrp=share-interactive,elapse=00:10:00      # Aquarius 1GPU
pjsub --interact -g <グループ> -L rscgrp=interactive-a,elapse=00:10:00          # Aquarius 1ノード(8GPU)
```

インタラクティブ用のノードが空いていないと、空くまで待たされる。`rscgrp` は `rg` と略記できる。

## ジョブスクリプト雛形

**Odyssey（MPI + OpenMP、1ノードあたり1プロセス × 48スレッド）**

```bash
#!/bin/bash
#PJM -L rscgrp=regular-o   # リソースグループ
#PJM -L node=12            # ノード数
#PJM --mpi proc=12         # MPI プロセス総数（flat MPI なら 12×48=576）
#PJM --omp thread=48       # OpenMP スレッド数
#PJM -L elapse=01:00:00    # 経過時間の上限
#PJM -g gz00               # グループ（必須）
#PJM -j                    # 標準エラーを標準出力にまとめる

module load odyssey
mpiexec ./a.out
```

**Aquarius（share 系：GPU 単位で確保）**

```bash
#!/bin/bash
#PJM -L rscgrp=share       # share / share-short / share-debug
#PJM -L gpu=1              # GPU 数（1, 2, 4）
#PJM -L elapse=02:00:00
#PJM -g gz00
#PJM -j

cd $PJM_O_WORKDIR
module load aquarius cuda
python train.py
```

**Aquarius（ノード単位で 8GPU を確保）**

```bash
#!/bin/bash
#PJM -L rscgrp=regular-a   # regular-a / short-a / debug-a
#PJM -L node=1
#PJM -L elapse=12:00:00
#PJM -g gz00
#PJM -j

cd $PJM_O_WORKDIR
module load aquarius cuda
torchrun --nproc_per_node=8 train.py   # 実行するコマンドは一例
```

- share 系は `-L gpu=N`、ノード単位のグループ（`*-a`、`*-o`）は `-L node=N` で指定する
- 作業は `/work/<グループ>/<ユーザ>` で行う（下記「ストレージ」参照）
- よく使う環境変数: `$PJM_O_WORKDIR`（投入時のディレクトリ）、`$PJM_JOBID`、`$PJM_MPI_PROC`、`$PJM_O_NODEINF`（ノードリストのファイル）

**Singularity コンテナを使う場合**は `#PJM -L jobenv=singularity` が**必須**:

```bash
#PJM -L rscgrp=share
#PJM -L gpu=1
#PJM -L jobenv=singularity
#PJM -g gz00
module load singularity
singularity exec --nv --bind $(pwd) image.sif python train.py
```

## リソースグループ

投入時に指定するのは次の名前だけ。`small-o`・`share-1` などの内部名は**直接指定できない**（ノード数や GPU 数によって自動で振り分けられる）。

**Odyssey**（メモリは 28GiB / ノード）

| rscgrp | ノード数 | 経過時間上限 |
|---|---|---|
| `debug-o` | 1〜144 | 30分 |
| `short-o` | 1〜72 | 8時間 |
| `regular-o` | 1〜2,304 | 48時間（1,153ノード以上は24時間） |
| `priority-o` | 1〜288 | 48時間（トークンの消費係数 1.5倍） |
| `interactive-o` | 1 / 2〜12 | 30分 / 10分 |

**Aquarius（ノード単位、8GPU / ノード、448GiB）**

| rscgrp | ノード数 | 経過時間上限 |
|---|---|---|
| `debug-a` | 1 | 30分 |
| `short-a` | 1〜2 | 2時間 |
| `regular-a` | 1〜8 | 48時間（5ノード以上は24時間） |
| `interactive-a` | 1 | 10分 |

**Aquarius（GPU 単位の share、1GPU ごとに CPU 9コア・56GiB）**

| rscgrp | GPU数 | 経過時間上限 |
|---|---|---|
| `share-debug` | 1, 2, 4 | 30分 |
| `share-short` | 1, 2, 4 | 2時間 |
| `share` | 1, 2, 4 | 48時間（4GPU は24時間） |
| `share-interactive` | 1 | 10分 |

ほかに、ログインノードと同じ構成の `prepost`（1ノード、6時間、340GiB）がある。ポータルから予約できる。
制限値は改定されることがある。最新は https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/service/job.php を確認する。

## トークン（課金）

```
トークン = 経過時間[h] × ノード数（Aquarius は GPU 数） × 消費係数
```

| 対象 | 消費係数 |
|---|---|
| Odyssey | 1.00 / ノード（`priority-o` は 1.50） |
| Aquarius | 3.00 / GPU（ノード単位のグループは 1ノード = 8GPU として計算） |

例: Aquarius の `regular-a` で 2ノード × 24時間 = 24 × 16 × 3.00 = 1,152 トークン。
- 残量は `show_token`
- トークンが切れても、ログインとインタラクティブジョブは使える（バッチジョブだけ投入できなくなる）
- 年度をまたいだ繰り越しはできない。Miyabi との間でトークンを移すことはできる

## モジュール

```bash
module avail                         # 使えるモジュール一覧
module list                          # ロード中のモジュール
module load odyssey                  # Odyssey 用の環境（富士通コンパイラ・MPI）
module load aquarius                 # Aquarius 用の環境
module load gcc cuda ompi-cuda       # Aquarius: GCC + CUDA + CUDA 対応の Open MPI
module load nvidia cuda ompi-cuda    # Aquarius: NVIDIA HPC SDK（nvc/nvfortran, OpenACC）
module load singularity
```

- Aquarius 向けの OpenACC: `nvc -O3 -acc -Minfo=accel -gpu=cc80`（A100 = compute capability 8.0）
- 入っているバージョンは時期によって変わるので `module avail` で確認する
- Python 環境として Miniconda / Archiconda が提供されている。Odyssey（aarch64）と Aquarius（x86_64）は CPU アーキテクチャが違うので、同じ環境は使い回せない

## ストレージ

| パス | 用途 | 注意 |
|---|---|---|
| `/home/<ユーザ>` | 設定ファイル・ソースコード | 容量や計算ノードから見えるかは公開情報では不明。**ジョブの入出力には使わない** |
| `/work/<グループ>/<ユーザ>` | 作業・ジョブの入出力 | 共有ファイルシステム（FEFS 25.8PB）。**ジョブはここから投入する** |
| `/work/<グループ>/share` | グループ内で共有 | |
| 高速ファイルシステム | 高 I/O 用 | FEFS 1.0PB / 1TB/s（パスは利用手引書で確認） |
| Ipomoea-01 | センター共通の大規模ストレージ | Wisteria と Miyabi の両方からアクセスできる。別途申し込みが必要 |

- `/work` のクォータはグループ単位で、申込セット数によって決まる（例: 4セットなら10TB）。ファイル数は 1TB あたり 50万個まで
- `/work/<グループ>` は初期状態で `2750`。他のグループに見せたいときはグループ代表者が `setfacl -m g:<別グループ>:rx .` を実行する

## 利用制限（主なもの）

| 項目 | Odyssey | Aquarius |
|---|---|---|
| 同時に使えるノード／GPU の上限 | 2,304ノード | 64GPU |
| 同時実行ジョブ数 | グループの申込セット数で決まる | グループの申込セット数で決まる |

実際の上限は `pjstat --limit` で確認する。

## よくあるエラー

| 症状 | 対処 |
|---|---|
| `qsub` / `sbatch`: command not found | Wisteria は PJM。`pjsub` を使う |
| グループ未指定で投入エラー | `#PJM -g <グループ>` か `pjsub -g <グループ>` を付ける |
| `cannot execute binary file` / `Exec format error` | Odyssey 用（aarch64）とログインノード・Aquarius 用（x86_64）のバイナリを取り違えている。Odyssey 向けは `*px` のクロスコンパイラで作る |
| `small-o` / `share-1` を指定して拒否される | 内部名は指定できない。`regular-o` / `share` を指定する |
| ジョブが始まらない | `pjstat --rscuse`（混雑）→ `pjstat --limit`（グループの同時実行上限）の順に確認 |
| バッチジョブが投入できない | `show_token` でトークン残量を確認 |
| ジョブ内でファイルが見つからない | `/home` ではなく `/work/<グループ>/<ユーザ>` に置き、そこから投入する |
| Singularity が動かない | `#PJM -L jobenv=singularity` を付けたか確認 |
| share で 3GPU を要求して拒否される | share 系の GPU 数は 1, 2, 4 のいずれか |

## 他システムとの混同注意

| システム | 関係 | 注意 |
|---|---|---|
| Miyabi | 同センターの後発機。2026年9月時点で Wisteria と並行して運用中 | 別システム。Wisteria のスクリプトをそのまま使わない。トークンは相互に移せる |
| Oakbridge-CX / Oakforest-PACS / Reedbush | 旧機 | ジョブスクリプトに**互換性がない**と公式に明記されている |

東大センターの Web 記事は、Oakforest-PACS・Oakbridge-CX・Reedbush 時代のものも多く残っている。リソースグループ名が `*-o` / `*-a` / `share*` でない記事は、別システム向けと判断する。
