# Contributing Guide

## Commit Message Guidelines

このリポジトリでは [Conventional Commits](https://www.conventionalcommits.org/) に基づくコミットメッセージを採用しています。

### Format

```text
<type>(<scope>): <subject>

<body>

```

- Header（1 行目）: 日本語で記述する。
- Body（3 行目以降）:
  - 変更の理由や背景、詳細な内容を日本語で記述する。
  - 複数の変更が含まれる場合は、箇条書き（ハイフン - 使用）を用いて整理することを推奨する。 日本語で詳細な理由や背景を記述する。
- Scope（省略可）: 変更が影響するコンポーネント名（例: `alloy`, `docker`, `rpi`）。

### Types

| Type     | Description                                |
| -------- | ------------------------------------------ |
| feat     | 新機能の追加                               |
| fix      | バグ修正                                   |
| docs     | ドキュメントのみの変更                     |
| style    | コードの動作に影響しないフォーマットの変更 |
| refactor | バグ修正や機能追加を含まないコードの変更   |
| perf     | パフォーマンスを向上させる変更             |
| test     | テストの追加・修正                         |
| chore    | ビルドプロセスやドキュメント生成などの雑用 |
