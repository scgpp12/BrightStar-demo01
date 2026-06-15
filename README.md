# BrightStar / 受託開発ケイパビリティ 紹介サイト

クラウド・生成AI・業務システムの受託開発ケイパビリティを紹介する静的サイトです。

## 構成
- `index.html` — 受託開発ケイパビリティ紹介（ランディングページ）
- `brightstar.html` — 実績「BrightStar 研修アシスタント」の詳細ページ
- `docs/` — 関連ドキュメント
  - `教员使用手册.pdf` — 教員向け操作マニュアル
  - `BrightStar_受託開発ケーススタディ.pdf` — ご紹介プレゼン資料
  - `功能说明.md` / `使用说明.md` / `架构与代码逻辑.md` / `系统要求.md`

`index.html` の「BrightStar 研修アシスタント」カードから `brightstar.html` にリンクしています。

## 閲覧・公開
- ローカル：`index.html` をブラウザで開く（または `python -m http.server`）
- 公開：GitHub Pages を有効化、または Vercel / Cloudflare Pages に静的サイトとしてデプロイ

技術：HTML + Tailwind CSS (CDN) + Font Awesome + Noto Sans JP。ビルド不要の静的サイト。
