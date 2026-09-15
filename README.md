# hpc-skills

各種 HPC 環境向けの [Claude Code](https://code.claude.com/) スキル集。

```bash
npx skills add ohtaman/hpc-skills --skill tsubame4
```

## スキル一覧

| スキル名 | 対象 | 説明 |
|---|---|---|
| `tsubame4` | TSUBAME4.0 (東京科学大学) | 汎用リファレンス |
| `tsubame4-minicamp` | TSUBAME4.0 ミニキャンプイベント | イベント固有の差分設定 |
| `sakuracloud` | さくらのクラウド / 高火力（さくらインターネット） | 汎用リファレンス |

## スキルの分け方

### 汎用スキル: `<system>`

公式ドキュメントをベースにした恒久的なリファレンス。誰でも・いつでも使える内容のみ収録する。

- ログイン・SSH 設定
- ジョブスクリプトのテンプレート
- 資源タイプ・モジュール・ストレージの仕様
- 制限値・禁止事項

### イベント固有スキル: `<system>-<event>`

ハンズオン・講習会・ミニキャンプなど、**特定イベント期間中のみ有効なルール**を汎用スキルの差分として記録する。

- イベント用グループ名・ARID
- 使用可能なキュー・予約枠の期間
- 共有ストレージのパス
- 参加者間の作法

汎用スキルと合わせて使う想定。イベント固有スキルだけで完結させない。

### イベントが終わったら

```bash
# 1. その回のスナップショットをタグで保存
git tag minicamp-tsubame4-20260911

# 2. SKILL.md のイベント情報を次回向けに更新（不明な項目は ???? に戻す）

# 3. push
git push origin main --tags
```

過去イベントの設定を参照したいときは、タグを URL に含めてインストール:

```bash
npx skills add https://github.com/ohtaman/hpc-skills/tree/minicamp-tsubame4-20260911/skills/tsubame4-minicamp
```

## インストール

```bash
# 個別インストール（プロジェクトローカル）
npx skills add ohtaman/hpc-skills --skill tsubame4

# グローバルインストール
npx skills add ohtaman/hpc-skills --skill tsubame4 -g

# 全スキルを一括インストール
npx skills add ohtaman/hpc-skills --all
```

## 新しい HPC 環境を追加するには

```
skills/
  <system>/
    SKILL.md          # 汎用リファレンス（チートシート）
    references/
      handbook.md     # 詳細ドキュメント
  <system>-<event>/   # イベント固有（任意）
    SKILL.md
```

`SKILL.md` のフロントマターの `name` はディレクトリ名と一致させる（必須）。
`SKILL.md` は 500 行以内を目安に、詳細は `references/` に分離する。
