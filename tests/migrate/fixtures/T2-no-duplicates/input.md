<!-- carta:begin v0.3.0 sha=4a06f48 -->
# carta-applied placeholder

このテキストはマーカー内 sha 検証用のプレースホルダ。
migrate は REF として実 assets/constitution.md を使うため、
ここの内容は重複検出には影響しない。
<!-- carta:end -->

# myrepo 開発ガイド

本リポジトリは TypeScript + React + Vite で書かれている。

## ビルドとテスト

- ローカル開発: `bun run dev` で Vite dev server を起動。
- ユニットテスト: `bun test`。
- e2e: `playwright test --project=chromium`。

## デプロイ

- main ブランチへの push で自動的に staging へデプロイされる（GitHub Actions）。
- prod は手動承認が必要（Slack の #release-approval で 1 名以上の approve）。

## ディレクトリ構成

- `src/components/` — UI コンポーネント
- `src/hooks/` — カスタム hook
- `src/api/` — API クライアント

## 関連リンク

- 設計ドキュメント: https://example.com/design
- 障害対応 runbook: https://example.com/runbook
