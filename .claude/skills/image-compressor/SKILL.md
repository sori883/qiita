---
name: image-compressor
description: CLI画像変換・圧縮ツール（cwebp, pngquant, jpegoptim）を使った画像の最適化。画像を圧縮したい、WebPに変換したい、PNG/JPEGを軽くしたい、画像を最適化したい、といったリクエスト時に使用する。
---

# Image Compressor

CLIツールを使って画像を変換・圧縮する。

## 前提条件

以下のツールが必要。未インストールの場合は `brew install` で導入する。

| ツール | 用途 | インストール |
|--------|------|-------------|
| cwebp | WebP変換 | `brew install webp` |
| pngquant | PNG圧縮 | `brew install pngquant` |
| jpegoptim | JPEG圧縮 | `brew install jpegoptim` |

## WebP変換（cwebp）

PNG/JPEGからWebPへ変換する。cwebp は入力ファイルを変更せず、新しい `.webp` を別ファイルとして書き出す（元ファイルは保持される）。

```bash
cwebp input.png -o output.webp
```

主要オプション:
- `-q <0-100>` — 品質指定（デフォルト75）
- `-lossless` — ロスレス圧縮
- `-resize <width> <height>` — リサイズ同時実行
- `-mt` — マルチスレッド

品質値の目安:
- `-q 75` — 標準（デフォルト）
- `-q 85` — 高品質（写真向き）
- `-q 90+` — 最高品質（圧縮率は下がる）
- `-lossless` — UI スクリーンショット・ロゴ等のテキストや鋭いエッジを完全保持したい場合

一括変換（カレントディレクトリのみ）:
```bash
for f in *.png; do cwebp -q 85 -mt "$f" -o "${f%.png}.webp"; done
```

サブディレクトリも含めて再帰的に変換:
```bash
find ./path -type f -name '*.png' -exec sh -c 'cwebp -q 85 -mt "$1" -o "${1%.png}.webp"' _ {} \;
```

## PNG圧縮（pngquant）

PNGを非可逆圧縮する。汎用ツールより劣化が少ない。

```bash
pngquant --quality=80-90 input.png
```

主要オプション:
- `--quality=<min>-<max>` — 品質範囲指定
- `--output <file>` — 出力先指定
- `--force` — 上書き許可
- `--skip-if-larger` — 圧縮後にサイズが増える場合スキップ
- `--strip` — メタデータ除去

一括圧縮:
```bash
pngquant --quality=80-90 *.png
```

## JPEG圧縮（jpegoptim）

JPEGを圧縮する。ロスレスにも対応。デフォルトでは入力ファイルを **上書き** するため、元を残す場合は `--dest` を指定する。

```bash
jpegoptim --strip-all --max=80 input.jpg
```

主要オプション:
- `--max=<0-100>` — 最大品質（指定しないとロスレス）
- `--strip-all` — 全メタデータ除去
- `--size=<n>` — ファイルサイズ目標（例: `--size=100k`）。指定時は目標サイズ達成のため品質を自動調整する（`--max` と併用すると `--max` が品質上限として効く）
- `--dest=<dir>` — 出力ディレクトリ指定。**指定先ディレクトリは事前に作成しておくこと**（例: `mkdir -p compressed && jpegoptim --dest=compressed ...`）

品質値の目安:
- `--max=80` — 標準（写真の web 表示十分）
- `--max=85` — 高品質
- 指定なし — ロスレス（サイズ削減幅は小さい）

一括圧縮:
```bash
jpegoptim --strip-all --max=80 *.jpg
```

元ファイルを残しつつサイズ目標で圧縮（複合パターン）:
```bash
mkdir -p ./compressed
jpegoptim --strip-all --size=100k --dest=./compressed input.jpg
```

## 使い分け

| 入力 | 目的 | コマンド |
|------|------|---------|
| PNG → WebP | 次世代フォーマット変換 | `cwebp input.png -o output.webp` |
| PNG → PNG（軽量化） | PNG維持で圧縮 | `pngquant --quality=80-90 input.png` |
| JPEG → JPEG（軽量化） | JPEG維持で圧縮 | `jpegoptim --strip-all --max=80 input.jpg` |
| JPEG → WebP | 次世代フォーマット変換 | `cwebp input.jpg -o output.webp` |
