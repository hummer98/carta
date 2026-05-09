# carta

統一的な開発憲法 (constitution) を保持し、各プロジェクトの `CLAUDE.md` に適用・同期する Claude Code Plugin。

> 🚧 現在は scaffold 段階。実装は `docs/seed.md` を参照して別セッションで進行中。

## コンセプト

- **carta** はラテン語/イタリア語で「憲章」を意味する（Magna Carta の "carta"）
- 全プロジェクトに共通して適用したい開発原則を 1 箇所で管理し、各リポジトリの `CLAUDE.md` に同期する
- フレームワークやプロジェクト性質によるカスタマイズ層は保ちつつ、共通原則部分だけを差分管理する

## インストール（予定）

```
/plugin marketplace add hummer98/carta
/plugin install carta
```

## 使い方（予定）

```
/carta:apply       # 現在のプロジェクトの CLAUDE.md に憲法を適用 / 同期
/carta:diff        # 適用前後の差分をプレビュー
/carta:show        # 現在の憲法本体を表示
```

詳細は `docs/seed.md` を参照。

## License

[MIT](LICENSE)
