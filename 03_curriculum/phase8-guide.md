# Phase 8 手順書：環境変数管理

> **ゴール：** 全環境変数がローカル・Vercel・ドキュメントで正しく管理されている状態にする  
> **所要時間の目安：** 1〜1.5時間  
> **前提：** Phase 7 が完了していること

---

## エージェント向け指示

このドキュメントの各Stepには実施者ラベルが付いています。

- **`[USER]`** — ユーザーが手動で実施する作業（ダッシュボード操作・アカウント作成など）。エージェントはこのStepの「ユーザーへの提示内容」をユーザーにそのまま提示し、完了確認を待ってから次のStepに進むこと。
- **`[AGENT]`** — エージェントが実施する作業（コード実装・コマンド実行など）。エージェントが自律的に実行すること。

---

## このPhaseで学ぶこと

- **シークレット管理の原則** — 公開していい変数とダメな変数の区別
- **`NEXT_PUBLIC_` プレフィックスの意味と危険性** — つけていい変数と絶対につけてはいけない変数
- **本番と開発の環境分離** — Direct connection vs Transaction mode の違い
- **`.env.example` によるオンボーディング改善** — 新しい開発者がすぐ始められる仕組み
- **環境変数の「棚卸し」習慣** — リリース前に必ず確認するチェックポイント

---

## このPhaseで使うツール

| ツール         | 役割                       | 料金       |
| -------------- | -------------------------- | ---------- |
| Vercel環境変数 | 本番環境のシークレット管理 | 無料枠あり |

> **コラム：Dopplerとは**
>
> [Doppler](https://www.doppler.com/) は環境変数を一元管理する専用SaaS。開発・ステージング・本番の環境変数をひとつのダッシュボードで管理でき、チームメンバー間でのシークレット共有やローテーション（定期的なキー更新）にも対応している。個人プロジェクトでは永久無料枠がある。
>
> このカリキュラムではデプロイ先がVercel一本のため、Vercelの組み込み環境変数マネージャーで十分対応できる。Dopplerのような専用ツールが必要になるのは以下のようなケースだ。
>
> - **複数のデプロイ先**がある（Vercel + AWS Lambda + Cloud Run など）
> - **チーム開発**で複数人がシークレットにアクセスする必要がある
> - **シークレットのローテーション**を自動化したい
>
> 「こういうツールがある」と知っておくことが重要。必要になったときに思い出せればよい。

---

## このPhaseで実施すること

- [ ] `.env.example` とコードベースを突き合わせて環境変数を棚卸し
- [ ] Vercelの環境変数が全件設定されているか確認・補完
- [ ] `.gitignore` でシークレットが除外されているか検証
- [ ] `.env.example` を最新状態に更新
- [ ] READMEの環境変数セクションを最新化
- [ ] 本番環境で全機能の最終動作確認

---

## 環境変数の分類について

このPhaseで最も重要な概念。どの変数を公開してよく、どの変数を秘密にすべきかの判断基準を理解すること。

| 分類         | プレフィックス      | ブラウザに公開 | 例                                                               |
| ------------ | ------------------- | -------------- | ---------------------------------------------------------------- |
| 公開可能     | `NEXT_PUBLIC_` あり | **される**     | `NEXT_PUBLIC_SUPABASE_URL`、`NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` |
| サーバー専用 | `NEXT_PUBLIC_` なし | **されない**   | `SUPABASE_SERVICE_ROLE_KEY`、`STRIPE_SECRET_KEY`、`DATABASE_URL` |

**絶対ルール：** `SUPABASE_SERVICE_ROLE_KEY`、`STRIPE_SECRET_KEY`、`DATABASE_URL`、`STRIPE_WEBHOOK_SECRET`、`SENTRY_AUTH_TOKEN` には `NEXT_PUBLIC_` を**絶対につけない**。つけるとブラウザ経由で誰でも見られる状態になり、データベースの全データ漏洩や不正決済につながる。

---

## Step 1 `[AGENT]`：環境変数の棚卸し

### やること

`.env.example` とコードベース内の `process.env.*` 参照箇所を突き合わせ、Phase 1〜7で追加された全環境変数を一覧化する。漏れや不整合がないか確認する。

### なぜ棚卸しをするのか

Phase 1〜7を進めるうちに環境変数が増えていく。途中で追加した変数が `.env.example` に反映されていなかったり、使われなくなった変数が残っていたりする。リリース前にこれを整理しておかないと、新しい環境へのデプロイ時に「動かない」トラブルが発生する。

### 確認手順

1. `.env.example` の全変数を一覧化する
2. コードベース内で `process.env.` を grep し、実際に使われている変数を一覧化する
3. 両者を突き合わせ、以下を確認する
   - `.env.example` にあるがコードで使われていない変数はないか
   - コードで使われているが `.env.example` にない変数はないか
4. 各変数について以下を整理する

### 全環境変数リファレンス

| 変数名                               | 用途                                    | 追加Phase | 公開/非公開 |
| ------------------------------------ | --------------------------------------- | --------- | ----------- |
| `NEXT_PUBLIC_SUPABASE_URL`           | Supabase プロジェクトURL                | Phase 1   | 公開可      |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY`      | Supabase 公開キー                       | Phase 1   | 公開可      |
| `SUPABASE_SERVICE_ROLE_KEY`          | Supabase 秘密キー（サーバーサイドのみ） | Phase 1   | **非公開**  |
| `DATABASE_URL`                       | Prisma用 Postgres接続文字列             | Phase 1   | **非公開**  |
| `UPLOADTHING_TOKEN`                  | UploadThing APIトークン                 | Phase 5   | **非公開**  |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Stripe 公開可能キー                     | Phase 6   | 公開可      |
| `STRIPE_SECRET_KEY`                  | Stripe シークレットキー                 | Phase 6   | **非公開**  |
| `NEXT_PUBLIC_APP_URL`                | アプリのベースURL                       | Phase 6   | 公開可      |
| `STRIPE_WEBHOOK_SECRET`              | Stripe Webhook署名シークレット          | Phase 6   | **非公開**  |
| `NEXT_PUBLIC_POSTHOG_KEY`            | PostHog プロジェクトAPIキー             | Phase 7   | 公開可      |
| `NEXT_PUBLIC_POSTHOG_HOST`           | PostHog インジェストURL                 | Phase 7   | 公開可      |
| `SENTRY_AUTH_TOKEN`                  | Sentry ビルド用認証トークン             | Phase 7   | **非公開**  |

> **注意：** Sentryの DSN はウィザードが設定ファイルに直接書き込むため、環境変数としては管理しない。

### 確認ポイント

- `.env.example` とコードベースの間に漏れ・不整合がないこと
- 各変数の公開/非公開が正しいこと（`NEXT_PUBLIC_` の有無）

---

## Step 2 `[USER]`：Vercel環境変数の確認・補完

### やること

Vercelダッシュボードで全環境変数が設定されているか確認し、不足分を追加する。

### なぜ重要なのか

Vercelに環境変数が設定されていないと、ローカルでは動くのに本番では動かないという状況になる。Phase 1〜7で段階的に追加してきたため、途中で追加し忘れているケースがある。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> **① Vercelダッシュボードで環境変数を確認する**
>
> Vercelダッシュボード →「Settings」→「Environment Variables」を開き、以下の全変数が設定されているか確認する。
>
> | 変数名                               | 確認ポイント                                                                        |
> | ------------------------------------ | ----------------------------------------------------------------------------------- |
> | `NEXT_PUBLIC_SUPABASE_URL`           | Supabase プロジェクトURLと一致しているか                                            |
> | `NEXT_PUBLIC_SUPABASE_ANON_KEY`      | Supabase Publishable keyと一致しているか                                            |
> | `SUPABASE_SERVICE_ROLE_KEY`          | Supabase Secret keyと一致しているか                                                 |
> | `DATABASE_URL`                       | **Transaction モード**の接続文字列になっているか（`pooler.supabase.com:6543`）      |
> | `UPLOADTHING_TOKEN`                  | UploadThingダッシュボードのToken（SDK v7+タブ）と一致しているか                     |
> | `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Stripe公開可能キー（`pk_test_...`）と一致しているか                                 |
> | `STRIPE_SECRET_KEY`                  | Stripeシークレットキー（`sk_test_...`）と一致しているか                             |
> | `NEXT_PUBLIC_APP_URL`                | VercelのデプロイURL（例：`https://my-folio.vercel.app`）になっているか              |
> | `STRIPE_WEBHOOK_SECRET`              | Stripeダッシュボードの本番用Webhookエンドポイントの署名シークレットと一致しているか |
> | `NEXT_PUBLIC_POSTHOG_KEY`            | PostHogプロジェクトAPIキー（`phc_...`）と一致しているか                             |
> | `NEXT_PUBLIC_POSTHOG_HOST`           | `https://us.i.posthog.com` または `https://eu.i.posthog.com`                        |
> | `SENTRY_AUTH_TOKEN`                  | Sentryの認証トークンと一致しているか                                                |
>
> **② 不足している変数があれば追加する**
>
> 不足分を追加した場合は、追加後に**再デプロイが必要**（Vercelは環境変数の追加後に自動で再デプロイしない）。
>
> **③ `DATABASE_URL` の接続モードを再確認する**
>
> > ⚠️ **重要：** Vercel の `DATABASE_URL` には `.env.local` と同じ Direct connection 文字列をそのまま使ってはいけない。Direct はVercelのサーバーレス環境から接続できないため、必ず **Transaction** モード（`pooler.supabase.com:6543`）の接続文字列を使うこと。
>
> 完了したら教えてください。

### 完了確認

- Vercelの環境変数に全12変数が設定されていること
- `DATABASE_URL` が Transaction モードの接続文字列になっていること
- 不足分を追加した場合、再デプロイが完了していること

---

## Step 3 `[AGENT]`：`.gitignore` と Git履歴の検証

### やること

`.gitignore` でシークレットファイルが除外されていることを確認し、Git履歴にシークレットが混入していないかチェックする。

### なぜ検証するのか

`.env.local` が `.gitignore` に含まれていても、過去に誤ってコミットした履歴が残っていれば、GitHubの履歴から誰でもシークレットを閲覧できる。「今は除外されている」だけでは不十分で、「過去にも入っていない」ことを確認する必要がある。

### 検証手順

1. `.gitignore` に以下のパターンが含まれているか確認する
   - `.env*` または `.env.local`
   - `.env.sentry-build-plugin`（Sentryウィザードが生成するファイル）
2. Git履歴に `.env.local` が含まれていないか確認する

```bash
git log --all --diff-filter=A -- .env.local
```

出力が空であれば問題なし。出力がある場合は、シークレットが漏洩した可能性があるため、該当するサービスでAPIキーを再発行する必要がある。

3. コードベース内にAPIキーがハードコードされていないか確認する

```bash
grep -r "sk_test_\|sk_live_\|whsec_\|service_role" --include="*.ts" --include="*.tsx" --include="*.js" .
```

Sentryの DSN（`https://xxxxx@o12345.ingest.us.sentry.io/67890`）が設定ファイルにハードコードされているのは正常（ウィザードの仕様）。それ以外のキーがコードに含まれていればNG。

### 確認ポイント

- `.gitignore` に `.env*` パターンが含まれていること
- `.gitignore` に `.env.sentry-build-plugin` が含まれていること
- Git履歴に `.env.local` が含まれていないこと
- コードにAPIキーがハードコードされていないこと（Sentry DSN は例外）

---

## Step 4 `[AGENT]`：`.env.example` の最終更新

### やること

`.env.example` を確認し、Phase 1〜7で追加された全環境変数が網羅されているか検証する。不足があれば追加する。

### 要件

- 全変数のキー名を記載する（値は**空のまま**）
- 各変数にコメントで用途と取得元を記載する
- Phase ごとにセクション分けする
- `NEXT_PUBLIC_` がつく変数とつかない変数が正しいか確認する

### `.env.example` の理想形

```bash
# ============================================
# Phase 1: Supabase + Prisma
# ============================================

# Supabase（Phase 1 で取得）
# supabase.com → Project Settings → API Keys
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Prisma（Supabase Postgres 接続）
# ローカル開発用：Direct connection の接続文字列を使う
# Vercel本番用：Transaction モードの接続文字列を使う
DATABASE_URL=

# ============================================
# Phase 5: UploadThing
# ============================================

# uploadthing.com → API Keys → SDK v7+ タブから取得
UPLOADTHING_TOKEN=

# ============================================
# Phase 6: Stripe
# ============================================

# stripe.com → API Keys から取得（サンドボックスは pk_test_ / sk_test_）
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
STRIPE_SECRET_KEY=
# ローカル: http://localhost:3000 / 本番: Vercel のデプロイ URL
NEXT_PUBLIC_APP_URL=
# ローカル: stripe listen で取得した whsec_... / 本番: Stripe ダッシュボードのエンドポイント詳細から取得
STRIPE_WEBHOOK_SECRET=

# ============================================
# Phase 7: PostHog + Sentry
# ============================================

# posthog.com → Settings → Project → General から取得（phc_ で始まる）
NEXT_PUBLIC_POSTHOG_KEY=
# US Cloud: https://us.i.posthog.com / EU Cloud: https://eu.i.posthog.com
NEXT_PUBLIC_POSTHOG_HOST=

# Sentry の DSN はウィザードが設定ファイルに直接書き込むため、ここには含めない
# SENTRY_AUTH_TOKEN は .env.sentry-build-plugin に保存される
```

### エージェントへの指示例

> `.env.example` を確認し、上記の理想形と比較して不足している変数やコメントがあれば追加してください。値は空のままにしてください。

### 確認ポイント

- Phase 1〜7 の全環境変数が `.env.example` に記載されていること
- 各変数にコメントで用途と取得元が書かれていること
- 値がすべて空であること（シークレットが含まれていないこと）

---

## Step 5 `[AGENT]`：READMEの環境変数セクションを更新する

### やること

`README.md` の環境変数セクションを最新状態に更新する。Phase 1〜7 で追加された全変数をカバーする。

### 要件

- 全変数のキー名と用途を一覧表で記載する
- **値は絶対に書かない**
- サーバー専用の変数にはその旨を明記する
- `.env.example` をコピーして使う手順を記載する

### READMEに記載すべき内容

````markdown
## 必要な環境変数

`.env.example` をコピーして `.env.local` を作成し、値を設定してください。

\```bash
cp .env.example .env.local
\```

| 変数名                               | 用途                                                  | 取得元                      |
| ------------------------------------ | ----------------------------------------------------- | --------------------------- |
| `NEXT_PUBLIC_SUPABASE_URL`           | Supabase プロジェクト URL                             | Supabase ダッシュボード     |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY`      | Supabase 公開キー（anon）                             | Supabase ダッシュボード     |
| `SUPABASE_SERVICE_ROLE_KEY`          | Supabase 秘密キー（サーバーサイドのみ）               | Supabase ダッシュボード     |
| `DATABASE_URL`                       | Prisma 用 Postgres 接続文字列                         | Supabase Connect            |
| `UPLOADTHING_TOKEN`                  | UploadThing API トークン                              | uploadthing.com             |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Stripe 公開可能キー                                   | Stripe ダッシュボード       |
| `STRIPE_SECRET_KEY`                  | Stripe シークレットキー（サーバーサイドのみ）         | Stripe ダッシュボード       |
| `NEXT_PUBLIC_APP_URL`                | アプリのベース URL                                    | 手動設定                    |
| `STRIPE_WEBHOOK_SECRET`              | Stripe Webhook 署名シークレット（サーバーサイドのみ） | Stripe CLI / ダッシュボード |
| `NEXT_PUBLIC_POSTHOG_KEY`            | PostHog プロジェクト API キー                         | PostHog ダッシュボード      |
| `NEXT_PUBLIC_POSTHOG_HOST`           | PostHog インジェスト URL                              | PostHog ダッシュボード      |

> ⚠️ `SUPABASE_SERVICE_ROLE_KEY`・`STRIPE_SECRET_KEY`・`DATABASE_URL`・`STRIPE_WEBHOOK_SECRET` は絶対に GitHub に公開しないこと。
````

### エージェントへの指示例

> `README.md` の環境変数セクションを上記の内容に更新してください。Phase実装状況の Phase 8 にもチェックを入れてください。

### 確認ポイント

- READMEに全環境変数が一覧で記載されていること
- 値が一切含まれていないこと
- `.env.example` をコピーする手順が書かれていること
- サーバーサイド専用の変数にその旨が明記されていること

---

## Step 6 `[USER]`：本番環境の最終動作確認とマージ

### やること

Vercel本番URLで全機能が正常に動作することを確認し、featureブランチをPR→mainにマージする。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> **① featureブランチをpushする**
>
> ```bash
> git add .
> git commit -m "chore: 環境変数管理を整備"
> git push origin feature/env-management
> ```
>
> GitHubでPRを作成してmainにマージする。
> VercelダッシュボードでデプロイがSuccessになるまで待つ。
>
> **② 本番URLで全機能の動作確認をする**
>
> 以下をすべて確認する。
>
> | 確認項目           | 確認方法                                                      |
> | ------------------ | ------------------------------------------------------------- |
> | トップページの表示 | 本番URLにアクセスし、作品一覧が表示されること                 |
> | 認証               | `/admin/login` からログインできること                         |
> | 作品のCRUD         | 管理画面から作品の作成・編集・削除ができること                |
> | 画像アップロード   | 作品投稿フォームから画像をアップロードできること              |
> | 決済               | 有料作品の「購入する」ボタンからStripe Checkoutに遷移すること |
> | エラー監視         | Sentryダッシュボードにプロジェクトが表示されていること        |
> | アナリティクス     | PostHogダッシュボードにページビューが記録されること           |
>
> **③ 不具合があった場合**
>
> 不具合がある場合は、まずVercelの環境変数を確認すること。ほとんどの本番環境の不具合は環境変数の設定漏れが原因。
>
> 完了したら教えてください。

### 完了確認

- Vercelのデプロイが成功していること
- 本番URLで全機能が正常に動作すること
- featureブランチがmainにマージされていること

---

## Phase 8 完了チェックリスト

- [ ] `.env.example` に Phase 1〜7 の全環境変数が記載されていること
- [ ] `.env.example` の各変数にコメントで用途と取得元が書かれていること
- [ ] Vercelの環境変数に全変数が設定されていること
- [ ] Vercelの `DATABASE_URL` が Transaction モードの接続文字列になっていること
- [ ] `.gitignore` に `.env*` と `.env.sentry-build-plugin` が含まれていること
- [ ] Git履歴に `.env.local` が含まれていないこと
- [ ] コードにAPIキーがハードコードされていないこと（Sentry DSN は例外）
- [ ] `README.md` に全環境変数の一覧が記載されていること（値は含まない）
- [ ] 本番URL（Vercel）で全機能が動作すること
- [ ] featureブランチで開発してPR→mainにマージした

---

## よくある失敗パターン

**本番で特定の機能だけ動かない**
Vercelに環境変数を追加し忘れている。とくに Phase 5（UploadThing）・Phase 6（Stripe）・Phase 7（PostHog）で追加した変数が漏れやすい。Vercelダッシュボードの「Environment Variables」で全件確認すること。また、環境変数を追加した後は**再デプロイが必要**。

**Vercelの `DATABASE_URL` で接続エラーが出る**
`.env.local` の Direct connection 文字列をそのままVercelにコピーしている。Vercelのサーバーレス環境では Direct connection（`db.xxx.supabase.co:5432`）は接続できない。Supabase Connect → **Transaction** モード（`pooler.supabase.com:6543`）の接続文字列に差し替えること。

**`NEXT_PUBLIC_` を秘密キーにつけてしまった**
`SUPABASE_SERVICE_ROLE_KEY` や `STRIPE_SECRET_KEY` に `NEXT_PUBLIC_` をつけると、ブラウザのJavaScriptから誰でもキーを取得できる状態になる。発見したら**すぐにキーを無効化し、各サービスのダッシュボードで再発行する**こと。

**`.env.local` をGitHubにコミットしてしまった**
`git rm` で削除しても**Git履歴には残る**。キーを再発行するしか対処法はない。Supabase・Stripe・UploadThing・PostHog の全サービスでAPIキーを再発行し、`.env.local` と Vercel の環境変数を更新すること。

**Sentryのビルドエラー（`SENTRY_AUTH_TOKEN`）**
Sentryウィザードは `.env.sentry-build-plugin` にトークンを保存するが、Vercelデプロイ時にはこのファイルが存在しない。Vercelの環境変数に `SENTRY_AUTH_TOKEN` を追加する必要がある。

---

## 全Phase完了 🎉

Phase 8が完了したら、Folio研修カリキュラムは全Phase完了です。

ここまでで体験したことを振り返ると、以下のツール・サービスを使って **月額固定費 $0** でアプリをリリースできる状態になっています。

| Phase | ツール                       | 役割                         |
| ----- | ---------------------------- | ---------------------------- |
| 1     | Next.js・Supabase・Vercel    | フレームワーク・DB・デプロイ |
| 2     | Supabase Auth                | 認証                         |
| 3     | Tailwind CSS・shadcn/ui      | UI                           |
| 4     | Prisma・Zod・React Hook Form | フォーム・DB操作             |
| 5     | UploadThing                  | ファイルアップロード         |
| 6     | Stripe                       | 決済                         |
| 7     | Sentry・PostHog・Lighthouse  | 監視・分析・パフォーマンス   |
| 8     | Vercel環境変数               | シークレット管理             |

> _「リリースされた不完全なものは、磨かれたままローンチされないものに毎回勝る」_
>
> ツールを使え。エコシステムを信頼しろ。速くリリースしろ。リアルなユーザーから学べ。

---

_このガイドはFolio研修カリキュラムの一部です。不明点はメンターに確認してください。_
