# AGENTS.md

このリポジトリはVibe CodingのベストプラクティスをFolioアプリの実装を通じて研修するためのガイダンスリポジトリです。

---

## リポジトリ構成

```
vibe-coding-best-practice-guide/
├── README.md                        # リポジトリ概要（人間向け）
├── AGENTS.md                        # このファイル（エージェント向け指示書）
│
├── 01_principles/
│   └── vibe-coding-guide-ja.md      # DO/DONTのベストプラクティス原則
│
├── 02_app-spec/
│   └── folio-spec.md                # Folioアプリの仕様書（技術スタック・データモデル・画面設計）
│
└── 03_curriculum/
    ├── phase1-guide.md              # Phase1手順書（デプロイ基盤）
    ├── phase2-guide.md              # Phase2手順書（認証）※今後追加
    └── ...
```

---

## 各ファイルの読む順序

アプリの実装タスクを受けた場合、以下の順番でファイルを参照すること。

1. `01_principles/vibe-coding-guide-ja.md` — 何を使い、何を作らないかの原則を把握する
2. `02_app-spec/folio-spec.md` — アプリの仕様・技術スタック・データモデルを把握する
3. `03_curriculum/phaseN-guide.md` — 該当Phaseの手順と意図を把握する

---

## 実装時の絶対ルール

以下は原則ファイルから抽出した、このプロジェクトで絶対に守るべきルール。

- **認証は自作しない** → Supabase Auth を使う
- **生のCSSは書かない** → Tailwind CSS + shadcn/ui を使う
- **REST APIをゼロから作らない** → Server Actions を使う
- **ファイルアップロードを自作しない** → UploadThing を使う
- **決済システムを自作しない** → Stripe を使う
- **APIキーをコードにハードコードしない** → 環境変数（.env.local）を使う
- **mainブランチに直接pushしない** → featureブランチ → PR → merge
- **完璧を目指してリリースを遅らせない**

---

## 技術スタック（確定）

| カテゴリ | ツール |
|---|---|
| フレームワーク | Next.js 14（App Router・TypeScript） |
| スタイリング | Tailwind CSS + shadcn/ui |
| DB / ORM | Supabase（Postgres） + Prisma |
| 認証 | Supabase Auth |
| ファイルアップロード | UploadThing |
| 決済 | Stripe Checkout |
| エラー監視 | Sentry |
| 分析 | PostHog |
| デプロイ | Vercel |

スタックの変更・追加は仕様書（`02_app-spec/folio-spec.md`）を確認してから行うこと。

---

## フォルダ構造（実装アプリ側）

実装するFolioアプリのフォルダ構造は `02_app-spec/folio-spec.md` のセクション7を参照すること。

---

## Phaseカリキュラムの概要

| Phase | テーマ | 主なツール |
|---|---|---|
| 1 | デプロイ基盤 | Next.js・Supabase・Vercel |
| 2 | 認証 | Supabase Auth |
| 3 | UI構築 | Tailwind・shadcn/ui |
| 4 | フォーム・DB | Prisma・Zod・React Hook Form・Server Actions |
| 5 | ファイルアップロード | UploadThing |
| 6 | 決済 | Stripe |
| 7 | 監視・分析 | Sentry・PostHog・Lighthouse |
| 8 | 環境変数管理 | Vercel環境変数 |

各Phaseの詳細は `03_curriculum/phaseN-guide.md` を参照すること。
