---
name: tsubame4
description: TSUBAME4.0（東京科学大学スーパーコンピュータ）の操作・ジョブ投入を行うときに使う。ログイン(ssh)、バッチジョブ(qsub/qstat/qdel)、インタラクティブジョブ(iqrsh)、ジョブスクリプト作成、資源タイプ(node_f/node_q/gpu_1等)、moduleコマンド、ストレージ(/home /gs/bs /gs/fs)、H100 GPU、MPI/OpenMP並列計算。「tsubame」「t4」「スパコン」「qsub」「ジョブを投げる」「UGE」「Grid Engine」等が出てきたら参照。詳細は references/handbook.md。
---

# TSUBAME4.0

東京科学大学 情報基盤センターのスーパーコンピュータ。
ジョブスケジューラ: **Altair Grid Engine (UGE)**（Slurm ではない。`sbatch`/`squeue` は使えない）

詳細リファレンス → `references/handbook.md`
公式ドキュメント → https://www.t4.cii.isct.ac.jp/docs/handbook.ja/

## ログイン

```bash
ssh <account>@login.t4.gsic.titech.ac.jp -i ~/.ssh/t4-key -YC
```

~/.ssh/config 設定例:

```
Host tsubame4
    HostName login.t4.gsic.titech.ac.jp
    User <account>
    IdentityFile ~/.ssh/t4-key
    ForwardX11 yes
    Compression yes
```

認証は SSH 公開鍵のみ（パスワード不可）。鍵はポータルで事前登録。
ログインノード: GPU なし、10分超の処理は禁止。

## ジョブ投入

```bash
qsub -g <グループ> job.sh          # バッチジョブ投入
qstat                               # ジョブ一覧 (r=実行中, qw=待機, Eqw=エラー)
qstat -j <ジョブID>                  # 詳細・エラー原因確認
qdel <ジョブID>                      # 削除（-f で強制）
qacct -j <ジョブID>                  # 終了後の実績・課金確認
```

インタラクティブジョブ:

```bash
iqrsh -l h_rt=1:00:00                               # 専用キュー (node_o 相当、グループ不要)
qrsh -g <グループ> -l node_q=1 -l h_rt=1:00:00     # 通常キュー
```

## ジョブスクリプト雛形

```bash
#!/bin/sh
#$ -cwd                   # カレントディレクトリで実行
#$ -l gpu_1=1             # 資源タイプ=個数（必須）
#$ -l h_rt=1:00:00        # Wall time（必須）
#$ -N myjob               # ジョブ名
#$ -j y                   # エラーを標準出力にマージ

module purge
module load cuda
./a.out
```

`-g <グループ>` はスクリプト内ではなく `qsub` の引数で渡す。

## 資源タイプ

| 資源タイプ | CPUコア | メモリ | GPU | 備考 |
|---|---|---|---|---|
| `node_f` | 192 | 768GB | H100×4 | フルノード（SSH ログイン可） |
| `node_h` | 96 | 384GB | H100×2 | ハーフノード |
| `node_q` | 48 | 192GB | H100×1 | クォーターノード |
| `node_o` | 24 | 96GB | H100×0.5 (MIG) | 1/8 ノード |
| `gpu_1` | 8 | 96GB | H100×1 | GPU 1枚重視 |
| `gpu_h` | 4 | 48GB | H100×0.5 (MIG) | GPU 半分 (MIG) |
| `cpu_160`〜`cpu_4` | 160〜4 | 368〜9GB | なし | CPU のみ |

- 異種混在不可（同一タイプを複数個のみ: `-l node_f=4`）
- `node_f` 確保時のみ計算ノードへ SSH ログイン可

## モジュール

```bash
module avail                        # 利用可能一覧
module list                         # ロード中を確認
module purge                        # 全クリア（スクリプト冒頭で必ず実行）
module load cuda                    # CUDA 最新版
module load cuda/12.8.0             # バージョン指定
module load intel                   # Intel oneAPI (icx/icpx/ifx + MKL)
module load nvhpc                   # NVIDIA HPC SDK（cuda と排他）
module load intel-mpi               # Intel MPI
module load openmpi/5.0.7-intel     # OpenMPI + Intel（cuda も自動ロード）
module load openmpi/5.0.7-gcc       # OpenMPI + GCC
```

Intel コンパイラ: `icx`(C) / `icpx`(C++) / `ifx`(Fortran)（`icc`/`ifort` は非推奨）

## ストレージ

| パス | 種別 | 容量目安 | 注意 |
|---|---|---|---|
| `/home/0/<account>` | — | 125GiB（ワークと合算） | 超過で書き込み不可 |
| `/gs/fs/<グループ>` | SSD/Lustre | グループ購入 | 高速アクセス向け |
| `/gs/bs/<グループ>` | HDD/Lustre | グループ購入 | 大容量データ向け |
| `${T4TMPDIR}` | ノードローカル SSD | 最大 1.62TiB | **ジョブ終了時に自動削除** |

```bash
t4-user-info disk home      # ホーム使用量
t4-user-info disk group     # グループディスク使用量
t4-user-info group point    # TSUBAMEポイント残量
```

- `/tmp` は使用禁止。一時ファイルは `${T4TMPDIR}` を使う
- Lustre 上で小ファイルを大量作成しない（HDF5 / WebDataset 等にまとめる）

## 利用制限

| 項目 | 平日 | 土日祝 |
|---|---|---|
| 同時実行ジョブ数 | 30 | 100 |
| 最大並列度 | 64ノード | 64ノード |
| 最大 Wall Time | 24時間 | 24時間 |

## よくあるエラー

| 症状 | 対処 |
|---|---|
| `Eqw` | `qstat -j <ID>` でエラー確認 |
| CRLF エラー | `dos2unix job.sh` |
| 権限エラー | `chmod u+rx job.sh` |
| ディスク超過 | `t4-user-info disk home` 確認後、不要ファイル削除 |
| `dr` のまま終わらない | `qdel -f <ID>` で強制削除 |
| ポイント不足 | `t4-user-info group point` 確認 |
