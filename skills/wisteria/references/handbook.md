# Wisteria/BDEC-01 詳細リファレンス

東京大学情報基盤センター「Wisteria/BDEC-01」（Big Data & Extreme Computing）。
公式サービスページ: https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/service/

本書は**公開されている**情報の要点をまとめたもの。正式な「Wisteria/BDEC-01 システム利用手引書」
（バッチジョブ・インタラクティブジョブ・ジョブ状態表示などの章を含む）は、ログインが必要な利用支援ポータル
https://wisteria-www.cc.u-tokyo.ac.jp/ にしかない。数値・制限値は改定されることがあるので、重要な判断の前には
公式ページか手引書で確認すること。

出典の凡例:
- **[公式]** cc.u-tokyo.ac.jp の公式ページ
- **[講習会]** センター主催の講習会資料（公式に公開されている PDF だが、講習用のグループやリソースグループを使っている）
- **[二次]** GitHub などの第三者資料
- **[未確認]** 公開情報では裏付けが取れなかったもの

調査日: 2026-09-26

---

## システム概要

出典: https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/system.php [公式]

| 項目 | Wisteria-O（Odyssey） | Wisteria-A（Aquarius） |
|---|---|---|
| 理論演算性能（合計） | 25.9 PFLOPS | 7.2 PFLOPS |
| ノード数 | 7,680 | 45 |
| 主記憶（合計） | 240.0 TiB | 36.5 TiB |
| インターコネクト | Tofu インターコネクト D | InfiniBand HDR (200Gbps) × 4 |
| トポロジー | 6次元メッシュ／トーラス | フルバイセクションの Fat Tree |

### ノード構成

**Odyssey**: FUJITSU Supercomputer PRIMEHPC FX1000
- CPU: A64FX × 1（48コア + アシスタントコア 2 または 4）、2.2GHz、3.3792 TFLOPS
- メモリ: 32GiB、帯域 1,024GB/s

**Aquarius**: FUJITSU Server PRIMERGY GX2570 M6
- CPU: Intel Xeon Platinum 8360Y（Ice Lake）× 2（36 + 36 コア）、2.4GHz、5.53 TFLOPS
- メモリ: 512GiB、帯域 409.6GB/s
- GPU: NVIDIA A100 × 8（SM 108、メモリ 40GiB/基、帯域 1,555GB/s、19.5 TFLOPS/基）
- CPU–GPU 間は PCIe Gen4 x16、GPU 間は NVLink × 12

### ストレージ

| 名称 | ファイルシステム | 容量 | 転送速度 |
|---|---|---|---|
| 共有ファイルシステム | FEFS（DDN SFA7990XE × 16） | 25.8PB | 504GB/s |
| 高速ファイルシステム | FEFS（DDN SFA400NVXE × 16） | 1.0PB | 1.0TB/s |

[公式] system.php

このほかに、センター共通の大規模ストレージ **Ipomoea-01**（2022年1月運用開始）があり、
Wisteria と Miyabi の両方からアクセスできる。申し込みは別
（https://www.cc.u-tokyo.ac.jp/supercomputer/ipomoea01/service/application.php ）。[公式／二次]

### ソフトウェア

[公式] system.php

- OS: Red Hat Enterprise Linux 8（Odyssey・Aquarius 共通）
- コンパイラ: GNU、富士通（Fortran/C/C++）、Intel（Fortran/C/C++）、NVIDIA HPC SDK（Fortran/C/C++/OpenACC）、CUDA
- MPI: 富士通 MPI、Intel MPI、Open MPI
- ライブラリ: 富士通 BLAS/LAPACK/ScaLAPACK、Intel MKL、cuBLAS/cuSPARSE/cuFFT/cuDNN/NCCL/MAGMA、FFTW、
  PETSc、Trilinos、METIS/ParMETIS、Scotch、SuperLU、HDF5、NetCDF、Boost、ppOpen-HPC、Xabclib ほか
- Python 環境: Miniconda、Archiconda
- アプリケーション: OpenFOAM、GROMACS、LAMMPS、Quantum ESPRESSO、CP2K、NWChem、ABINIT-MP、PHASE、FrontISTR、
  FrontFlow/blue、BLAST、ParaView、PyTorch、TensorFlow、Keras、Horovod、MXNet、Chainer、MATLAB、HyperWorks ほか
- コンテナ: Singularity

---

## ログイン・初期設定

### 接続

```bash
ssh -l <ユーザ名> wisteria.cc.u-tokyo.ac.jp
```

ホスト名 `wisteria.cc.u-tokyo.ac.jp` は第三者の資料で確認したもの
（https://github.com/matsuolab/wisteria_simple ）[二次]。公開されている公式ページにはホスト名が書かれていない。
ログインノードが複数あるかどうか、踏み台が必要かどうかは [未確認]（手引書で確認する）。

### 公開鍵の登録

[公式] https://www.cc.u-tokyo.ac.jp/faq/wisteria.php

1. 手元で鍵ペアを作る（例: `ssh-keygen -t ed25519`）
2. 利用支援ポータル https://wisteria-www.cc.u-tokyo.ac.jp/ にログインし、公開鍵を登録する
   - ポータルの初回ログインには、書面またはメールで届く初期パスワードを使う
   - 書面で受け取った人は書面の注記を、メールで受け取った人はメールにある URL からパスワードを再設定する
3. SSH はパスワード認証ではなく公開鍵認証だけ

注意:
- ログインシェルの既定は bash。`chsh` で変更できる [公式]
- `~/.ssh` ディレクトリのパーミッションを壊してログインできなくなると、利用者は自力で復旧できない（サポートに連絡する）。
  変更前にバックアップを取る [公式]

### ログインノードのアーキテクチャ

[講習会] 第205回「Wisteria実践」 https://www.cc.u-tokyo.ac.jp/events/lectures/205/20230608-2.pdf

- ログインノード: Intel Cascade Lake + AVX-512（x86_64）。Odyssey と Aquarius で共通
- Odyssey の計算ノード: A64FX（Armv8.2-A + SVE、aarch64）→ 命令セットが大きく違うので**クロスコンパイルが必要**
- Aquarius の計算ノード: Ice Lake（x86_64 + AVX-512）→ ログインノードと「ほぼ同じ」なのでクロスコンパイルは要らない

---

## コンパイル

### Odyssey（富士通コンパイラ）

[講習会] 第205回、第213回「MPI基礎」 https://www.cc.u-tokyo.ac.jp/events/lectures/213/20231004-2.pdf

```bash
module load odyssey      # または module load fj（富士通 MPI は自動でロードされる）
```

| 言語 | ネイティブ（計算ノード上） | クロス（ログインノード上） |
|---|---|---|
| C | `fcc` | `fccpx` |
| C++ | `FCC` | `FCCpx` |
| Fortran | `frt` | `frtpx` |
| MPI | `mpi` + コンパイラ名（例 `mpifcc`） | `mpi` + コンパイラ名（例 `mpifccpx`） |

Makefile の例 [講習会]:

```make
CC     := mpifccpx
CFLAGS := -Kfast -Koptmsg=2
```

- `-Kfast`: 富士通コンパイラの標準的な最適化セット
- ネイティブコンパイラは、インタラクティブジョブなどで計算ノードに入っているときに使う

### Aquarius（GPU）

[講習会] 第178回「GPUプログラミング入門」、第205回、第213回

| 言語 | GNU | Intel | NVIDIA（旧 PGI） | CUDA |
|---|---|---|---|---|
| C | `gcc` | `icc` | `nvc`（`pgcc`） | `nvcc` |
| C++ | `g++` | `icpc` | `nvc++`（`pgc++`） | — |
| Fortran | `gfortran` | `ifort` | `nvfortran`（`pgfortran`） | — |

```bash
module load aquarius
module load gcc cuda ompi-cuda        # GCC + CUDA + CUDA 対応 Open MPI
module load nvidia cuda ompi-cuda     # NVIDIA HPC SDK
module load nvidia cuda/11.2 ompi-cuda  # CUDA のバージョンを固定する例

nvc -O3 -acc -Minfo=accel -gpu=cc80 -c main.c           # OpenACC（A100 = cc80）
nvfortran -O3 -mp -acc -ta=tesla,cc80 -Minfo=accel -c main.f90
nvfortran -acc -ta=tesla,managed ...                     # Unified Memory を使う
```

講習会資料の 2023年2月時点のバージョン: NVIDIA HPC SDK 22.7、CUDA 11.4、ompi-cuda 4.1.4-11.4。
**古い情報**なので、実際に入っている版は `module avail` で確認する。

Intel コンパイラのモジュール名は [未確認]（`module avail` で確認する）。

---

## ジョブ実行（富士通 TCS / PJM）

### 基本コマンド

[公式] https://www.cc.u-tokyo.ac.jp/faq/wisteria.php / [講習会] 第213回

| コマンド | 用途 |
|---|---|
| `pjsub <スクリプト>` | バッチジョブを投入する |
| `pjsub -g <グループ> <スクリプト>` | グループを引数で指定して投入する |
| `pjstat` | 自分のジョブの状況（どのトークンで実行されたかも表示される） |
| `pjdel <ジョブID>` | ジョブを削除する |
| `pjstat -H` | 過去の投入履歴 |
| `pjstat --rsc` | 投入できるリソースグループ（キュー）一覧 |
| `pjstat --rsc -x` | リソースグループの詳細構成 |
| `pjstat --rsc -b` | リソースグループごとの投入ジョブ数 |
| `pjstat --rscuse` | 計算ノードの混み具合 |
| `pjstat --limit` | グループの同時投入数・同時実行数の上限 |
| `show_token` | トークン残量 [公式] token.php |

`show_rscgrp` のような専用コマンドは公開情報では確認できなかった。リソースグループは `pjstat --rsc` で調べる。

### pjstat --limit の出力例

[公式] FAQ

```
$ pjstat --limit
SYSTEM: Odyssey
PROJECT     ACCEPT       RUN  BULK_ACCEPT  BULK_RUN      NODE
gXX0        0/ 128    0/  16         0/ 8    0/  16   0/ 2304
SYSTEM: Aquarius
PROJECT     ACCEPT       RUN  BULK_ACCEPT  BULK_RUN     GPU
gXX0         0/  4    0/   2         0/ -    0/   -   0/ 64
```

`ACCEPT` は同時に投入できる数、`RUN` は同時に実行できる数、`NODE` / `GPU` は同時に使える資源量。

### pjstat --rscuse の出力例

[公式] FAQ

```
$ pjstat --rscuse
SYSTEM: Odyssey
RSCGRP                                                  Ratio Used/Total(Node)
debug-o/interactive-o         -------------------------    0%     0/ 768
short-o                       -------------------------    0%     0/ 192
regular-o                     *******************------   76%  4096/5376
priority-o                    ******-------------------   25%   288/1152
SYSTEM: Aquarius
RSCGRP                                                  Ratio Used/Total(Node)
debug-a/interactive-a         ************-------------   50%     1/  2
short-a                       ************-------------   50%     1/  2
regular-a                     *************************  100%    24/ 24
RSCGRP                                                  Ratio Used/Total(GPU)
share-debug/share-interactive -------------------------    0%     0/   8
share-short                   ********-----------------   31%     5/  16
share                         **********************---   86%    55/  64
```

### #PJM ディレクティブ

講習会資料と第三者資料で使われているもの:

| ディレクティブ | 意味 |
|---|---|
| `#PJM -L rscgrp=<名前>` | リソースグループ（`rg=` と略記できる） |
| `#PJM -L node=<N>` | ノード数 |
| `#PJM -L gpu=<N>` | GPU 数（Aquarius の share 系） |
| `#PJM -L elapse=HH:MM:SS` | 経過時間の上限 |
| `#PJM -L jobenv=singularity` | Singularity を使う（**必須**） |
| `#PJM -g <グループ>` | グループ（課金先） |
| `#PJM --mpi proc=<N>` | MPI プロセス総数 |
| `#PJM --omp thread=<N>` | OpenMP スレッド数 |
| `#PJM -j` | 標準エラーを標準出力にまとめる |

`-L` の項目はカンマでつなげられる（例 `-L rscgrp=interactive-o,node=12,elapse=01:00`）。

出力ファイル名の規則、メール通知、ステップジョブ・バルクジョブの書き方は、手引書で確認する [未確認]。

### 環境変数

講習会・第三者資料のスクリプトで使われているもの:

| 変数 | 内容 |
|---|---|
| `$PJM_O_WORKDIR` | ジョブを投入したディレクトリ |
| `$PJM_JOBID` | ジョブ ID |
| `$PJM_MPI_PROC` | MPI プロセス数 |
| `$PJM_O_NODEINF` | 割り当てノードのリストのファイル（`mpiexec -machinefile` に渡す） |

ほかの変数の網羅的な一覧は手引書にある [未確認]。

### インタラクティブジョブ

[講習会] 第205回（講習会では `–g` の全角ダッシュが混じっているので、コピペするときは半角 `-g` に直す）

```bash
# Odyssey
pjsub --interact -g <グループ> -L rscgrp=interactive-o,elapse=00:30:00           # 1ノード
pjsub --interact -g <グループ> -L rscgrp=interactive-o,node=12,elapse=00:10:00   # 2〜12ノード

# Aquarius
pjsub --interact -g <グループ> -L rscgrp=share-interactive,elapse=00:10:00       # 1GPU
pjsub --interact -g <グループ> -L rscgrp=interactive-a,elapse=00:10:00           # 1ノード(8GPU)
```

- インタラクティブ用のノードがすべて使われていると、空くまでログインできない [講習会]
- トークンを消費しない [公式] job.php
- 経過時間の上限は下記「リソースグループ」表を参照。講習会の例（`elapse=01:00`）は上限を超えている可能性があるので、上限内の値にする

---

## ジョブスクリプト例

講習会の例は `lecture-o` / `lecture-a` / `tutorial-a`（講習会専用）、グループ `gt00`（講習会用）を使っている。
通常の利用では `regular-o` などのリソースグループと、自分のグループ名に置き換える。

### Odyssey: flat MPI（12ノード × 48プロセス）

[講習会] 第205回

```bash
#!/bin/bash
#PJM -L rscgrp=regular-o
#PJM -L node=12
#PJM --mpi proc=576
#PJM -L elapse=00:10:00
#PJM -g gz00

module load odyssey
mpiexec ./hello
```

### Odyssey: MPI + OpenMP（1ノードあたり1プロセス × 48スレッド）

[講習会] 第205回

```bash
#!/bin/bash
#PJM -L rscgrp=regular-o
#PJM -L node=12
#PJM --mpi proc=12
#PJM --omp thread=48
#PJM -L elapse=00:10:00
#PJM -g gz00

module load odyssey
mpiexec ./hello_omp
```

### Aquarius: GPU（MPI なし）

[講習会] 第205回

```bash
#!/bin/bash
#PJM -L rscgrp=share
#PJM -L gpu=4
#PJM -L elapse=00:10:00
#PJM -g gz00

module load aquarius cuda
./a.out
```

### Aquarius: GPU + MPI（4GPU、1ノードに4プロセス）

[講習会] 第205回

```bash
#!/bin/bash
#PJM -L rscgrp=share
#PJM -L gpu=4
#PJM --mpi proc=4
#PJM -L elapse=00:10:00
#PJM -g gz00

module load aquarius cuda ompi-cuda
mpiexec -machinefile $PJM_O_NODEINF -n $PJM_MPI_PROC \
  -npernode 4 ./wrapper.sh ./a.out
```

`wrapper.sh` は、各プロセスに別々の GPU を割り当てるために講習会で配布されたスクリプト
（MPI のローカルランクから `CUDA_VISIBLE_DEVICES` を決める類のもの）。中身は講習会資料を参照。

### Aquarius: Singularity（1GPU）

[講習会] 第205回

```bash
#!/bin/bash
#PJM -L rscgrp=share
#PJM -L gpu=1
#PJM -g gz00
#PJM -L jobenv=singularity     # Singularity を使うときは必須
#PJM -L elapse=00:15:00
#PJM -j

cd $PJM_O_WORKDIR
module load gcc cuda singularity
SIF=/work/gz00/share/tf-image.sif
singularity exec --nv --bind `pwd` $SIF python keras-tf-mnist.py \
  >& keras-tf-mnist.log.$PJM_JOBID
```

### Aquarius: Singularity（ノード単位、第三者の例）

[二次] https://github.com/matsuolab/wisteria_simple

```bash
#PJM -L rscgrp=debug-a
#PJM -L node=1
#PJM -L elapse=0:10:00
#PJM -L jobenv=singularity
#PJM -j
module load singularity
```

```bash
pjsub -g gb20 run_vdm_train.sh
```

この資料には「作業ディレクトリは `/work/<グループ>/` 配下にする必要があり、ホームディレクトリからは投入できない」とある。

---

## リソースグループ

出典: https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/service/job.php [公式]

- 投入時に指定できるのは、下の表の太字の名前だけ。`small-o` や `share-1` などの内部名は**直接指定できない**
  （ノード数や GPU 数によって自動で振り分けられる）
- トークンが残っていれば、どのリソースグループにも投入できる

### インタラクティブ（トークン消費なし）

| rscgrp | 内部名 | 資源 | 経過時間上限 | メモリ |
|---|---|---|---|---|
| **interactive-o** | interactive-o_n1 | 1ノード | 30分 | 28GiB |
| **interactive-o** | interactive-o_n12 | 2〜12ノード | 10分 | 28GiB |
| **interactive-a** | — | 1ノード | 10分 | 448GiB |
| **share-interactive** | — | 1GPU | 10分 | 56GiB |

interactive-o_n1 の制限時間は 2022-08-02 に見直されている。

### Odyssey（バッチ）

| rscgrp | 内部名 | ノード数 | 経過時間上限 | メモリ |
|---|---|---|---|---|
| **debug-o** | — | 1〜144 | 30分 | 28GiB |
| **short-o** | — | 1〜72 | 8時間 | 28GiB |
| **regular-o** | small-o | 1〜144 | 48時間 | 28GiB |
| | medium-o | 145〜576 | 48時間 | 28GiB |
| | large-o | 577〜1,152 | 48時間 | 28GiB |
| | x-large-o | 1,153〜2,304 | 24時間 | 28GiB |
| **priority-o** | — | 1〜288 | 48時間 | 28GiB |

`priority-o` は優先利用ノード群（全体の約15%）で、消費係数が 1.50 になる。

### Aquarius（ノード単位、バッチ）

| rscgrp | 内部名 | ノード数 | 経過時間上限 | メモリ |
|---|---|---|---|---|
| **debug-a** | — | 1 | 30分 | 448GiB |
| **short-a** | — | 1〜2 | 2時間 | 448GiB |
| **regular-a** | small-a | 1〜2 | 48時間 | 448GiB |
| | medium-a | 3〜4 | 48時間 | 448GiB |
| | large-a | 5〜8 | 24時間 | 448GiB |

### Aquarius（GPU 単位の share、バッチ）

1GPU あたり CPU 9コア、メモリ 56GiB が割り当てられる。

| rscgrp | 内部名 | GPU 数 | 経過時間上限 |
|---|---|---|---|
| **share-debug** | — | 1, 2, 4 | 30分 |
| **share-short** | — | 1, 2, 4 | 2時間 |
| **share** | share-1 / share-2 / share-4 | 1 / 2 / 4 | 48時間（4GPU は24時間） |

### プリポスト

| rscgrp | ノード数 | 経過時間上限 | メモリ |
|---|---|---|---|
| **prepost** | 1 | 6時間 | 340GiB |

ログインノードと同じ構成で、対話的な前処理・後処理に使う。利用支援ポータルから予約することもできる。

### ノード固定・GPU 専有コース

Aquarius には、特定のノードや GPU を専有する申込コースがある
（https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/service/fixed_node.php ）。
専用のリソースグループ名や投入方法は [未確認]。

---

## 同時実行数などの上限

[公式] FAQ

| 項目 | Odyssey | Aquarius |
|---|---|---|
| ユーザ単位の上限 | なし | なし |
| グループの同時実行ジョブ数 | 申込セット数で決まる（1〜2セット: 4本、3〜4セット: 8本、5〜8セット: 16本 …） | 申込セット数で決まる（最大20本） |
| 同時に使える資源の上限 | 2,304ノード | 64GPU |

自分のグループの実際の値は `pjstat --limit` で確認する。

---

## トークン（課金）

出典: https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/service/token.php [公式]

```
トークン = 経過時間 × 要求ノード数（または GPU 数） × 消費係数
```

Aquarius のノード単位のリソースグループでは、1ノード = 8GPU として計算する。

| 対象 | 消費係数 |
|---|---|
| Odyssey | 1.00 / ノード |
| Odyssey 優先利用ノード群（`priority-o`） | 1.50 / ノード |
| Aquarius | 3.00 / GPU |

計算例 [公式]:
- Odyssey 128ノード × 24時間 = 24 × 128 × 1.00 = 3,072
- `priority-o` 128ノード × 24時間 = 24 × 128 × 1.50 = 4,608
- Aquarius 2ノード（16GPU） × 24時間 = 24 × 16 × 3.00 = 1,152

申込1セットあたりの年間割当:

| コース | トークン |
|---|---|
| 通常利用 | 8,640 |
| Aquarius ノード固定 | 207,360（1セットのみ） |
| Aquarius GPU 専有 | 25,920（1, 2, 4セットのみ） |

そのほかの規則:
- 残量は `show_token`、ジョブがどのトークンで実行されたかは `pjstat` で確認する
- トークンを使い切っても、ログインとインタラクティブジョブは使える（バッチジョブだけ投入できなくなる）
- 年度をまたいだ繰り越し、グループ間の移動、返金はできない。有効期限は利用期限の月末か、当該年度の年度末サービス終了日
- トークンは Miyabi と Wisteria/BDEC-01 の間で移すことができる（https://www.cc.u-tokyo.ac.jp/guide/application/ ）[公式]

---

## ストレージとファイル共有

### /work のクォータ

[公式] FAQ

- `/work/<グループ>/<ユーザ>` と `/work/<グループ>/share` に、グループ内の割当量が設定される
- 例: 4セット/年の申し込みなら、ユーザ・グループとも 10TB（基本 2TB + 1セットあたり 2TB）
- ファイル・ディレクトリの数は、割当容量 1TB につき 50万個まで

### /home

公式ページには `/home` の容量や、計算ノードからアクセスできるかどうかが書かれていない [未確認]。
第三者の資料に「ホームディレクトリからはジョブを投入できない」とあるので [二次]、
ジョブの入出力はすべて `/work/<グループ>/<ユーザ>` に置く運用にする。

### 他のグループとの共有

[公式] FAQ

`/work/<グループ>` は初期状態でモード `2750`（他のグループからは読めない）。共有するときは、グループ代表者が ACL を設定する:

```bash
cd /work/<グループ>
setfacl -n -m u:<ユーザ名>:rx .
setfacl -n -m g:<別グループ>:rx .
getfacl <ファイル名>     # 設定を確認する
```

---

## トラブルシューティング

### ジョブが実行されない

[公式] FAQ の手順:

1. `pjstat --rscuse` でシステム全体の混み具合を確認する
2. `pjstat --limit` でグループの同時実行数の上限に達していないか確認する
3. どちらでもなければシステム障害の可能性がある → 問い合わせ窓口へ

### よくある間違い

| 症状 | 原因と対処 |
|---|---|
| Oakbridge-CX や Reedbush のスクリプトが通らない | Wisteria とは互換性がない。書き直す [公式] job.php |
| `Exec format error` / `cannot execute binary file` | Odyssey（aarch64）とログインノード・Aquarius（x86_64）のバイナリを取り違えている |
| 内部名（`small-o`、`share-1` など）での投入が拒否される | `regular-o` / `share` などの公開名を指定する |
| Singularity のジョブが動かない | `#PJM -L jobenv=singularity` を付ける |
| バッチジョブが投入できない | `show_token` でトークン残量を確認する |
| 講習会のスクリプトをそのまま投入して失敗する | `lecture-o` / `gt00` は講習会専用。自分のリソースグループとグループに置き換える |
| 講習会資料からコピーしたコマンドが失敗する | `–g`（全角ダッシュ）や `¥`（行継続）が混じっている。`-g`、`\` に直す |

---

## サービスの状況（2026-09-26 時点）

- センターのトップページ（https://www.cc.u-tokyo.ac.jp/ ）で Wisteria/BDEC-01 は「通常サービス中」[公式]
- 運用スケジュール（https://www.cc.u-tokyo.ac.jp/supercomputer/schedule.php ）には、月末処理・年度末処理の定期メンテナンス予定が載っている。
  恒久的なサービス終了の予告は見当たらない [公式]
- 2026年度も「大規模HPCチャレンジ」の募集が続いている（https://announce.cc.u-tokyo.ac.jp/announce/A01562.html ）[公式]
- 後発の Miyabi とは並行して運用されている。Wisteria の終了予定日や後継機（BDEC-02 など）の計画は、公開ページでは確認できなかった [未確認]
- 月末には定期メンテナンスでサービスが止まることがある。止まっているときはまず運用スケジュールを見る

---

## 出典一覧

- システム構成: https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/system.php
- ジョブクラス: https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/service/job.php
- トークン: https://www.cc.u-tokyo.ac.jp/supercomputer/wisteria/service/token.php
- FAQ: https://www.cc.u-tokyo.ac.jp/faq/wisteria.php
- 利用申込: https://www.cc.u-tokyo.ac.jp/guide/application/
- 運用スケジュール: https://www.cc.u-tokyo.ac.jp/supercomputer/schedule.php
- 講習会 第205回「Wisteria実践」: https://www.cc.u-tokyo.ac.jp/events/lectures/205/20230608-2.pdf
- 講習会 第213回「MPI基礎」: https://www.cc.u-tokyo.ac.jp/events/lectures/213/20231004-2.pdf
- 第三者の例: https://github.com/matsuolab/wisteria_simple
