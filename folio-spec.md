# Folio — クリエイター向け作品公開・販売サイト 仕様書

> **目的：** バイブコーディングのベストプラクティスを研修するためのサンプルアプリ  
> **対象：** デザイナー・イラストレーターが自分の作品を公開・販売する個人サイト  
> **方針：** 最初から最終形のスタックを導入し、Phaseごとに機能を追加していく（B案）

---

## 1. 技術スタック（確定）

| カテゴリ | ツール | 備考 |
|---|---|---|
| フレームワーク | Next.js 14（App Router） | TypeScript使用 |
| スタイリング | Tailwind CSS + shadcn/ui | 生CSS禁止 |
| DB / ORM | Supabase（Postgres） + Prisma | Prismaでスキーマ管理 |
| 認証 | Supabase Auth | オーナーのみログイン |
| ファイルアップロード | UploadThing | 作品画像1枚 |
| 決済 | Stripe Checkout | 購入フローのみ |
| エラー監視 | Sentry | |
| 分析 | PostHog | |
| パフォーマンス | Lighthouse | Chrome DevTools内蔵 |
| 環境変数 | Vercel環境変数 | .env.localと対応 |
| デプロイ | Vercel | GitHubと連携 |

---

## 2. ユーザー種別

| 種別 | 説明 | 認証 |
|---|---|---|
| **オーナー**（自分） | 作品を管理・投稿・販売設定する | ログイン必須 |
| **訪問者**（潜在顧客） | 作品を閲覧・購入する | 不要 |

---

## 3. 画面一覧

| 画面 | パス | 誰が見る | 説明 |
|---|---|---|---|
| トップ | `/` | 訪問者 | 作品一覧 + プロフィール |
| 作品詳細 | `/works/[id]` | 訪問者 | 画像・説明・価格・購入ボタン |
| 購入完了 | `/works/[id]/success` | 訪問者 | Stripe決済後のサンクスページ |
| ログイン | `/admin/login` | オーナー | Supabase Authログイン画面 |
| 管理ダッシュボード | `/admin` | オーナーのみ | 作品一覧・統計 |
| 作品追加 | `/admin/works/new` | オーナーのみ | 作品投稿フォーム |
| 作品編集 | `/admin/works/[id]/edit` | オーナーのみ | 作品編集フォーム |

---

## 4. データモデル

### Work（作品）

```prisma
model Work {
  id          String   @id @default(cuid())
  title       String
  description String
  category    Category
  imageUrl    String
  price       Int      // 0 = 無料、1以上 = 有料（円）
  published   Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

enum Category {
  ILLUSTRATION
  GRAPHIC
  UI
  OTHER
}
```

---

## 5. 機能仕様

### 5-1. 公開ページ（訪問者向け）

**トップページ**
- 公開済み作品（`published: true`）を一覧表示
- 作品カード：サムネイル画像・タイトル・カテゴリ・価格
- 作品が0件の場合は空状態メッセージを表示

**作品詳細ページ**
- 作品画像・タイトル・説明・カテゴリ・価格を表示
- 無料作品：「無料で見る」ボタン（外部リンクなど）
- 有料作品：「購入する」ボタン → Stripe Checkoutへ

**購入完了ページ**
- 「ありがとうございました」メッセージのみ
- トップに戻るリンク

### 5-2. 管理画面（オーナー向け）

**ログイン**
- メール + パスワード認証（Supabase Auth）
- 未ログイン状態で `/admin` にアクセスすると `/admin/login` にリダイレクト

**管理ダッシュボード**
- 作品一覧（公開・非公開含む全件）
- 各作品に「編集」「削除」ボタン
- 「作品を追加」ボタン

**作品投稿・編集フォーム**
- フィールド：タイトル・説明・カテゴリ・画像・価格・公開設定
- バリデーション（Zod）：
  - タイトル：必須、最大100文字
  - 説明：必須、最大1000文字
  - 価格：0以上の整数
  - 画像：必須
- 画像アップロード：UploadThing
- Server Actionsで送信

---

## 6. Phaseカリキュラム

### Phase 1：デプロイ基盤（Day 1〜2）
**ゴール：** 空のNext.jsアプリがVercelで動いている

- [ ] Next.js + TypeScript + Tailwind のプロジェクト作成
- [ ] shadcn/ui セットアップ
- [ ] Prisma + Supabase 接続設定
- [ ] `.env.local` + `.gitignore` 設定
- [ ] GitHub リポジトリ作成 + push
- [ ] Vercel デプロイ（環境変数設定含む）
- [ ] README 作成

**学ぶこと：** Vercelのpush-to-deploy、環境変数の扱い、プロジェクト構造

---

### Phase 2：認証（Day 2〜3）
**ゴール：** `/admin` にログインしないとアクセスできない

- [ ] Supabase Authでオーナーアカウント作成
- [ ] ログイン画面 `/admin/login` の実装
- [ ] middleware.ts で `/admin` 以下を保護
- [ ] ログアウト機能

**学ぶこと：** Supabase Auth、Next.js middleware、セッション管理

---

### Phase 3：UI構築（Day 3〜5）
**ゴール：** 公開ページと管理画面のUIが整っている

- [ ] トップページ（作品一覧）
- [ ] 作品詳細ページ
- [ ] 管理ダッシュボード（作品一覧）
- [ ] 空状態コンポーネント
- [ ] shadcn/ui コンポーネント使用（Button・Card・Badge・Input）

**学ぶこと：** shadcn/ui、Tailwind、Server Components vs Client Components

---

### Phase 4：状態管理・フォーム・DB（Day 5〜7）
**ゴール：** 作品のCRUDが動く

- [ ] Prisma スキーマ定義・マイグレーション
- [ ] 作品投稿フォーム（React Hook Form + Zod）
- [ ] 作品編集フォーム
- [ ] 削除確認ダイアログ
- [ ] Server Actions で DB 操作

**学ぶこと：** Prisma、Zod バリデーション、React Hook Form、Server Actions

---

### Phase 5：ファイルアップロード（Day 7〜8）
**ゴール：** 作品画像をアップロードできる

- [ ] UploadThing セットアップ
- [ ] 投稿・編集フォームに画像アップロードを追加
- [ ] アップロード済み画像のプレビュー表示

**学ぶこと：** UploadThing、ファイルアップロードのセキュリティ・CDN配信

---

### Phase 6：決済（Day 8〜9）
**ゴール：** 有料作品をStripeで購入できる

- [ ] Stripe セットアップ
- [ ] 「購入する」ボタン → Stripe Checkout セッション作成（Server Action）
- [ ] 購入完了ページ（`/works/[id]/success`）
- [ ] Stripe Webhook で購入記録（最小限）

**学ぶこと：** Stripe Checkout、Webhook の基本

---

### Phase 7：監視・分析（リリース前日）
**ゴール：** 本番で何が起きているか把握できる状態にする

- [ ] Sentry セットアップ・動作確認（意図的にエラーを起こして通知確認）
- [ ] PostHog セットアップ・ページビュー確認
- [ ] Lighthouse でパフォーマンス計測・スコア確認

**学ぶこと：** エラー監視の重要性、ユーザー行動分析、パフォーマンス改善

---

### Phase 8：環境変数管理（本番移行）
**ゴール：** 環境変数が正しく管理されている

- [ ] `.env.local` の全変数をVercelに設定されているか確認
- [ ] `.gitignore` に `.env*` が含まれているか確認
- [ ] README に必要な環境変数一覧を記載

**学ぶこと：** シークレット管理の原則、本番と開発の環境分離

---

## 7. フォルダ構造

```
my-folio/
├── app/
│   ├── (public)/           # 訪問者向けページ
│   │   ├── page.tsx        # トップ（作品一覧）
│   │   └── works/
│   │       └── [id]/
│   │           ├── page.tsx         # 作品詳細
│   │           └── success/
│   │               └── page.tsx     # 購入完了
│   ├── admin/              # オーナー向けページ（要認証）
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── page.tsx        # ダッシュボード
│   │   └── works/
│   │       ├── new/
│   │       │   └── page.tsx
│   │       └── [id]/
│   │           └── edit/
│   │               └── page.tsx
│   └── api/
│       └── uploadthing/    # UploadThingのAPIルート
├── components/
│   ├── ui/                 # shadcn/uiコンポーネント（自動生成）
│   ├── work-card.tsx       # 作品カード
│   ├── work-form.tsx       # 投稿・編集フォーム
│   └── empty-state.tsx     # 空状態
├── lib/
│   ├── supabase.ts         # Supabaseクライアント
│   ├── prisma.ts           # Prismaクライアント
│   ├── uploadthing.ts      # UploadThingクライアント
│   └── stripe.ts           # Stripeクライアント
├── actions/
│   ├── works.ts            # 作品のServer Actions
│   └── stripe.ts           # StripeのServer Actions
├── types/
│   └── index.ts            # 型定義
├── prisma/
│   └── schema.prisma       # DBスキーマ
├── middleware.ts            # 認証保護
├── .env.local              # 環境変数（gitignore対象）
├── .gitignore
└── README.md
```

---

## 8. 環境変数一覧

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Prisma
DATABASE_URL=

# UploadThing
UPLOADTHING_SECRET=
UPLOADTHING_APP_ID=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# App
NEXT_PUBLIC_APP_URL=

# Sentry
SENTRY_DSN=

# PostHog
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PUBLIC_POSTHOG_HOST=
```

---

## 9. DONTリスト（研修中に守ること）

- ❌ 認証をゼロから作らない → Supabase Auth を使う
- ❌ 生のCSSを書かない → Tailwind + shadcn/ui を使う
- ❌ REST APIをゼロから作らない → Server Actions を使う
- ❌ ファイルアップロードを自作しない → UploadThing を使う
- ❌ 決済システムを自作しない → Stripe を使う
- ❌ APIキーをコードにハードコードしない → .env.local を使う
- ❌ mainに直接pushしない → featureブランチ → PR → merge
- ❌ 完璧を目指してリリースを遅らせない

---

*作成日：2025年2月 / ベース資料：MVPを速く・ひとりで・確実にリリースするための完全ガイド*
