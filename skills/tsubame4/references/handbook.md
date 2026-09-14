# TSUBAME4.0 詳細リファレンス

出典: https://www.t4.cii.isct.ac.jp/docs/handbook.ja/

---

## システム概要

- 運用: 東京科学大学 情報基盤センター
- 計算ノード: HPE Cray XD665 × 240ノード
- 演算性能: 倍精度 66.8 PFLOPS / 半精度 952 PFLOPS
- OS: Red Hat Enterprise Linux 9.5
- ジョブスケジューラ: Altair Grid Engine 2023.1.1

### ノード仕様

| 項目 | ログインノード | 計算ノード |
|------|--------------|-----------|
| 台数 | 2 | 240 |
| CPU | AMD EPYC 7443 (24コア×2, 2.85GHz) | AMD EPYC 9654 (96コア×2, 2.4GHz) |
| メモリ | 256GB | 768GiB (DDR5-4800) |
| GPU | なし | NVIDIA H100 SXM5 × 4 |
| ローカル SSD | なし | 1.92TB NVMe U.2 |
| ノード間接続 | — | InfiniBand NDR200 200Gbps × 4 |

---

## ログイン・初期設定

### SSH 接続

```bash
# ホスト名（自動振り分け推奨）
ssh <account>@login.t4.gsic.titech.ac.jp -i ~/.ssh/t4-key -YC

# 固定接続
ssh <account>@login1.t4.gsic.titech.ac.jp
ssh <account>@login2.t4.gsic.titech.ac.jp
```

~/.ssh/config:

```
Host tsubame4
    HostName login.t4.gsic.titech.ac.jp
    User <account>
    IdentityFile ~/.ssh/t4-key
    ForwardX11 yes
    Compression yes
```

### 初回セットアップ

1. アカウント申請: https://www.t4.cii.isct.ac.jp/getting-account
2. ポータルで SSH 公開鍵を登録: https://portal.t4.gsic.titech.ac.jp/
3. `ssh login.t4.gsic.titech.ac.jp` で接続確認
4. グループ管理者がグループ作成・ストレージ購入・ポイント購入を実施
5. `t4-user-info group point` でポイント残量を確認

```bash
chsh   # ログインシェル変更（bash / tcsh / zsh）
```

---

## ジョブスケジューラ詳細

### コマンド一覧

| コマンド | 用途 |
|---------|------|
| `qsub` | バッチジョブ投入 |
| `qstat` | ジョブ状態確認 |
| `qdel` | ジョブ削除 |
| `qrsh` | インタラクティブジョブ（通常キュー） |
| `iqrsh` | インタラクティブジョブ（専用キュー） |
| `iqstat` | 専用キューのジョブ確認 |
| `iqdel` | 専用キューのジョブ削除 |
| `qacct` | ジョブ終了後の課金情報確認 |

### qsub オプション詳細

```bash
qsub -g <グループ> job.sh

-l <資源タイプ>=<個数>    # リソース指定（必須）
-l h_rt=HH:MM:SS        # Wall time（必須）
-N <名前>                # ジョブ名
-cwd                     # カレントディレクトリで実行
-o <ファイル>             # 標準出力先
-e <ファイル>             # 標準エラー出力先
-j y                     # エラー出力を標準出力にマージ
-m a|b|e                 # メール通知（a=中止, b=開始, e=終了）
-p -5|-4|-3              # 優先度（-5=標準, -3=最高。高いほどポイント消費大）
-ar <AR_ID>              # 予約ノード
-t 開始-終了[:ステップ]   # アレイジョブ
-hold_jid <ジョブ名/ID>  # 依存ジョブ（先行ジョブ完了後に実行）
```

### お試し実行（無償）

`-g` を省略すると無償の「お試し実行」になる。制限: 最大 2ノード / Wall time 3分 / 同時 1ジョブ。研究成果への使用は禁止。

---

## ジョブスクリプトテンプレート集

### シングル GPU

```bash
#!/bin/sh
#$ -cwd
#$ -l gpu_1=1
#$ -l h_rt=2:00:00
#$ -N gpu_job

module purge
module load cuda/12.8.0
./my_gpu_program
```

### OpenMP（SMP 並列）

```bash
#!/bin/sh
#$ -cwd
#$ -l node_f=1
#$ -l h_rt=1:00:00
#$ -N openmp_job

module purge
module load intel
export OMP_NUM_THREADS=192
./a.out
```

### MPI 並列（Intel MPI）

```bash
#!/bin/sh
#$ -cwd
#$ -l node_f=4
#$ -l h_rt=1:00:00
#$ -N mpi_job

module purge
module load cuda
module load intel
module load intel-mpi
# -ppn: 1ノードあたりのプロセス数, -n: 総プロセス数
mpiexec.hydra -ppn 8 -n 32 ./a.out
```

### MPI 並列（OpenMPI）

```bash
#!/bin/sh
#$ -cwd
#$ -l node_f=4
#$ -l h_rt=1:00:00
#$ -N openmpi_job

module purge
module load openmpi/5.0.7-intel   # cuda + intel も自動ロード
mpirun -npernode 8 -n 32 -x LD_LIBRARY_PATH ./a.out
```

### ハイブリッド並列（MPI + OpenMP）

```bash
#!/bin/sh
#$ -cwd
#$ -l node_f=4
#$ -l h_rt=1:00:00
#$ -N hybrid_job

module purge
module load cuda
module load intel
module load intel-mpi
export OMP_NUM_THREADS=192
mpiexec.hydra -ppn 1 -n 4 ./a.out
```

### アレイジョブ

```bash
#!/bin/sh
#$ -cwd
#$ -l cpu_4=1
#$ -l h_rt=1:00:00
#$ -N array_job
#$ -t 1-10:1          # タスク 1〜10

./a.out --input input_${SGE_TASK_ID}.dat
```

アレイジョブはタスク数が多いと待ち時間が増加する。1 タスクに複数処理をまとめることを推奨。

### ローカルスクラッチ活用

```bash
#!/bin/sh
#$ -cwd
#$ -l node_f=1
#$ -l h_rt=2:00:00

cp -rp $HOME/datasets ${T4TMPDIR}/
./a.out ${T4TMPDIR}/datasets ${T4TMPDIR}/results
cp -rp ${T4TMPDIR}/results $HOME/results
# ${T4TMPDIR} はジョブ終了時に自動削除
```

### バックグラウンド並列実行

```bash
./program1 &
./program2 &
wait   # wait がないとジョブが即終了する
```

---

## モジュール詳細

### 主要モジュール一覧

| 種別 | モジュール | 備考 |
|------|-----------|------|
| CUDA | `cuda`, `cuda/12.8.0` | — |
| Intel | `intel` | icx/icpx/ifx + MKL |
| NVIDIA HPC SDK | `nvhpc` | cuda と排他 |
| AMD | `aocc` | clang/flang |
| Intel MPI | `intel-mpi` | — |
| OpenMPI + GCC | `openmpi/5.0.7-gcc` | cuda/12.8.0 自動ロード |
| OpenMPI + Intel | `openmpi/5.0.7-intel` | intel + cuda 自動ロード |
| OpenMPI + NVHPC | `openmpi/5.0.7-nvhpc` | nvhpc 自動ロード |

`openmpi` はバージョン指定なしで使うと意図しないバージョンになる可能性があるため、**必ずバージョンを指定**する。

### Intel コンパイラのコマンド名

| 言語 | 推奨 | 非推奨（廃止予定） |
|------|------|--------------------|
| C | `icx` | `icc` |
| C++ | `icpx` | `icpc` |
| Fortran | `ifx` | `ifort` |

### 推奨コンパイルオプション（Intel）

```bash
icx -O3 -xCORE-AVX512 -o program program.c

# 2GB 超のグローバル変数がある場合
icx -mcmodel=medium -shared-intel -o program program.c
```

### コンテナ（Apptainer）

```bash
apptainer shell -B /gs -nv -f -w ubuntu/
# -B /gs: ストレージをマウント
# -nv: GPU（NVIDIA）を使用
```

---

## GPU 仕様（NVIDIA H100 SXM5）

| 項目 | 値 |
|------|-----|
| メモリ | 94GB HBM2e |
| メモリ帯域幅 | 2,395.87 GB/s |
| FP64 | 33.5 TFlops |
| FP64 Tensor | 66.9 TFlops |
| FP32 | 66.9 TFlops |
| TF32 Tensor | 494.7 TFlops |
| FP16 / BF16 Tensor | 989.4 TFlops |
| INT8 Tensor | 1,978.9 TOps |

通常の H100（80GB, 3.3TB/s）よりメモリが大きくやや帯域が遅い。

### MIG（Multi-Instance GPU）

`node_o` と `gpu_h` は MIG モード。複数ノード確保時、全ノードで MIG 初期化が必要:

```bash
module load openmpi
mpirun -np $(cat $PE_HOSTFILE | wc -l) -npernode 1 /bin/true
module purge
```

### GPU 確認コマンド

```bash
nvidia-smi
nvidia-smi --query-gpu=name,memory.total,pci.bus_id,mig.mode.current --format=csv
```

---

## MPI 詳細

### ノードファイル（PE_HOSTFILE）

```bash
cat $PE_HOSTFILE          # 割り当てノード一覧（hostname コア数 キュー名 <NULL>）
cat $PE_HOSTFILE | wc -l  # 割り当てノード数
```

### MPI 実行コマンド比較

| MPI | 実行コマンド | 1ノードあたり指定 |
|-----|------------|----------------|
| Intel MPI | `mpiexec.hydra` | `-ppn <N>` |
| OpenMPI | `mpirun` | `-npernode <N>` |

OpenMPI では環境変数の伝播に `-x LD_LIBRARY_PATH` が必要。

---

## ノード予約（Advance Reservation）

```bash
t4-user-info compute ar    # 自分の予約一覧
t4-user-info compute ars   # 当月の空き状況
qrstat -ar <AR_ID>         # 予約の詳細

# 予約ノードへのバッチジョブ
qsub -g <グループ> -ar <AR_ID> job.sh

# 予約ノードへのインタラクティブジョブ
qrsh -g <グループ> -l node_f=1 -l h_rt=2:00:00 -ar <AR_ID>
```

予約可能な資源タイプ: `node_f`, `node_h`, `node_q`, `node_o` のみ。

予約利用可能ノード数:
- 4月〜9月: 70ノード（最大予約期間 7日）
- 10月〜3月: 20ノード（最大予約期間 4日）

予約時のポイント消費は通常の 1.25〜10倍。

---

## 利用制限詳細

| 項目 | 平日 | 土日祝 |
|------|------|--------|
| 同時投入可能ジョブ数 | 2,000 | 2,000 |
| 同時実行ジョブ数 | 30 | 100 |
| CPU スロット上限 | 6,144 | 12,288 |
| 1ジョブあたり最大並列度 | 64ノード | 64ノード |
| 最大 Wall Time | 24時間 | 24時間 |

---

## 禁止事項・注意事項

### ログインノードで禁止

- 10分超・高負荷のプログラム実行
- VSCode Server / Jupyter Notebook の起動
- `cron` / `crontab`（システムで禁止）
- 大量ジョブの連続投入・`qstat` の頻繁な実行

### ジョブスクリプトの注意

- shebang（`#!/bin/sh`）を必ず先頭に記述
- スクリプト冒頭で `module purge` を実行
- `-V`（ログインノードの環境変数を引き継ぎ）は意図しない挙動の原因になるため使わない
- バックグラウンド実行後は必ず `wait` を入れる

### ストレージ注意

- ホーム + ワークの合計クォータは 125GiB。超過すると書き込み不可（削除後、再計算に最大 1日）
- `/tmp` は使用禁止。`${T4TMPDIR}` を使う
- Lustre 上に小ファイルを大量作成しない（HDF5 / WebDataset / TFRecord にまとめる）
- inode 上限: 2,000,000 / 1TB

### ポイント注意

- ジョブ投入時にポイント残高が自動チェックされる
- 優先度 `-p` を上げるとポイント消費増（-3 が最高消費）
- `t4-user-info group point` で定期確認

---

## 便利コマンド集

```bash
# ユーザ・グループ情報
t4-user-info group point         # TSUBAMEポイント残量
t4-user-info disk home           # ホーム・ワーク使用量
t4-user-info disk group          # グループディスク使用量
t4-user-info compute ar          # 自分の予約一覧
t4-user-info compute ars         # 当月の予約空き状況

# ジョブ管理
qsub -g <グループ> job.sh        # ジョブ投入
qstat                             # 自分のジョブ一覧
qstat -j <ID>                    # ジョブ詳細（エラー確認）
qdel <ID>                         # ジョブ削除
qdel -f <ID>                      # 強制削除（dr 状態に使用）
qacct -j <ID>                    # 終了後の実績確認

# モジュール
module avail                      # 利用可能一覧
module list                       # 現在のロード状況
module purge                      # 全モジュールをクリア

# GPU（計算ノード上）
nvidia-smi                        # GPU 状態確認
```

---

## 参照リンク

- [利用の手引き（日本語）](https://www.t4.cii.isct.ac.jp/docs/handbook.ja/)
- [ジョブ投入詳細](https://www.t4.cii.isct.ac.jp/docs/all/handbook.ja/jobs/)
- [FAQ: 一般](https://www.t4.cii.isct.ac.jp/docs/all/faq.ja/general/)
- [FAQ: ジョブスケジューラ](https://www.t4.cii.isct.ac.jp/docs/all/faq.ja/scheduler/)
- [ハードウェア構成](https://www.t4.cii.isct.ac.jp/hardware)
- [システムソフトウェア一覧](https://www.t4.cii.isct.ac.jp/system-software)
- [各種制限値](https://www.t4.cii.isct.ac.jp/resource-limit)
- [TSUBAME4.0 ポータル](https://portal.t4.gsic.titech.ac.jp/)
