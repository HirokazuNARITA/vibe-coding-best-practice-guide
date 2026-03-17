# Phase 7 手順書：監視・分析

> **ゴール：** 本番で何が起きているか把握できる状態にする  
> **所要時間の目安：** 2〜3時間  
> **前提：** Phase 6 が完了していること

---

## エージェント向け指示

このドキュメントの各Stepには実施者ラベルが付いています。

- **`[USER]`** — ユーザーが手動で実施する作業（ダッシュボード操作・アカウント作成など）。エージェントはこのStepの「ユーザーへの提示内容」をユーザーにそのまま提示し、完了確認を待ってから次のStepに進むこと。
- **`[AGENT]`** — エージェントが実施する作業（コード実装・コマンド実行など）。エージェントが自律的に実行すること。

---

## このPhaseで学ぶこと

- **エラー監視の重要性** — 本番で何が壊れたかをリアルタイムで検知する仕組み
- **ユーザー行動分析** — 誰がどのページを見ているかを把握する方法
- **パフォーマンス計測** — ページの表示速度やアクセシビリティを数値で評価する方法
- **「後から入れると手遅れ」** という原則の体験

---

## このPhaseで使うツール

| ツール     | 役割               | 料金                            |
| ---------- | ------------------ | ------------------------------- |
| Sentry     | エラー監視         | 無料（月5,000エラーまで）       |
| PostHog    | プロダクト分析     | 無料（寛大な無料枠）            |
| Lighthouse | パフォーマンス計測 | 完全無料（Chrome DevTools内蔵） |

---

## このPhaseで実装すること

- [ ] Sentryアカウント作成・プロジェクト作成
- [ ] Sentry SDKのセットアップ（`sentry.client.config.ts` / `sentry.server.config.ts` / `sentry.edge.config.ts`）
- [ ] 意図的なエラーを起こしてSentryに通知が届くことを確認
- [ ] PostHogアカウント作成・プロジェクト作成
- [ ] PostHog SDKのセットアップ（`PostHogProvider` の実装）
- [ ] ページビューが記録されることを確認
- [ ] Lighthouseでパフォーマンス計測・スコア確認
- [ ] Vercelの環境変数に追加

---

## なぜリリース前に入れるのか

監視・分析ツールは「問題が起きてから入れる」のでは遅い。

| タイミング         | 起きること                                                             |
| ------------------ | ---------------------------------------------------------------------- |
| リリース前に入れる | 初日からエラーを検知でき、ユーザー行動も把握できる                     |
| リリース後に入れる | 「なんか動かない」という報告が来て初めて気づく。原因調査に時間がかかる |

> 💡 Sentryがあれば「どのページで」「どのブラウザで」「どのエラーが」起きたかが即座にわかる。PostHogがあれば「誰もそのページを見ていない」こともわかる。

---

## Sentry とは

アプリで発生したエラーをリアルタイムで収集・通知してくれるサービス。Next.jsとの統合が公式にサポートされている。

| 機能             | 説明                                               |
| ---------------- | -------------------------------------------------- |
| エラー収集       | クライアント・サーバー両方のエラーを自動キャプチャ |
| スタックトレース | エラーが発生したコードの場所を特定                 |
| 通知             | メール・Slackなどでアラートを受信                  |
| リリース追跡     | どのデプロイでエラーが増えたかを追跡               |

---

## PostHog とは

ユーザーの行動を分析するためのオールインワンプラットフォーム。Google Analyticsの代替として使える。

| 機能             | 説明                                 |
| ---------------- | ------------------------------------ |
| ページビュー分析 | どのページがどれくらい見られているか |
| セッション録画   | ユーザーの操作をそのまま録画して再生 |
| ファネル分析     | 「トップ→詳細→購入」の離脱率を可視化 |
| A/Bテスト        | 2つのバージョンを比較して効果を検証  |

> 💡 このPhaseではページビュー分析のセットアップのみ行う。セッション録画やA/Bテストは必要になったときに有効化すればよい。

---

## Lighthouse とは

Googleが提供するWebページのパフォーマンス計測ツール。Chrome DevToolsに内蔵されており、インストール不要。

| カテゴリ       | 計測内容                                       |
| -------------- | ---------------------------------------------- |
| Performance    | ページの読み込み速度（LCP・FID・CLSなど）      |
| Accessibility  | スクリーンリーダーへの対応、コントラスト比など |
| Best Practices | HTTPSの使用、画像フォーマットの最適化など      |
| SEO            | メタタグ、構造化データ、モバイル対応など       |

---

## Step 1 `[USER]`：Sentryアカウントを作成してプロジェクトを作成する

### やること

Sentryにアカウントを作成し、Next.js用のプロジェクトを作成する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> **① アカウント作成・組織作成**
>
> 1. [sentry.io](https://sentry.io) にアクセスしてアカウントを作成する（GitHubログイン推奨）
> 2. 「Create a New Organization」画面が表示される
>    - **Organization Name：** 任意の名前を入力（例：`folio`）
>    - **Data Storage Location：** 「United States (US)」を選択
>    - Terms of Serviceに同意してチェックを入れる
> 3. 「Create Organization」をクリック
>
> **② オンボーディングをスキップする**
>
> 1. 「Welcome to Sentry」画面が表示されたら、左下の **「Skip Onboarding」** をクリック
>
> ⚠️ 14日間のBusinessトライアルが開始されるが、終了後は自動で無料プランに切り替わる。課金はされない。
>
> **③ プロジェクトを作成する**
>
> 1. ダッシュボードの **「Create project」** ボタンをクリック
> 2. 「Create a new project in 3 steps」画面が表示される
>    - **Step 1 - Choose your platform：** **「NEXT.JS」** を選択する
>    - **Step 2 - Set your alert frequency：** 「Alert me on high priority issues」のままでよい
>    - **Step 3 - Name your project and assign it a team：** 「Project slug」を入力する（例：`folio`）
> 3. **「Create Project」** をクリック
> 4. 「Configure Next.js SDK」画面が表示される
> 5. 画面に表示される `npx @sentry/wizard` コマンドをコピーする（Step 2で使用）
> 6. **「Copy DSN」** ボタンも押してDSNをメモしておく（参考用）
>
> ⚠️ この画面は閉じずに残しておくこと。Step 2でコマンドを使う。
>
> 完了したら教えてください。

### 完了確認

- Sentryにプロジェクトが作成されていること
- `npx @sentry/wizard` コマンドをコピーしていること

---

## Step 2 `[USER]`：Sentry SDKをセットアップする

### やること

Sentryウィザードを使ってNext.js SDKをセットアップする。ウィザードが設定ファイルを全部自動生成してくれる。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> **① ウィザードコマンドを実行する**
>
> Folioプロジェクトのルートディレクトリで、Step 1のセットアップ画面に表示されたコマンドを実行する。
>
> ```bash
> npx @sentry/wizard@latest -i nextjs --saas --org YOUR_ORG --project YOUR_PROJECT
> ```
>
> （`YOUR_ORG` と `YOUR_PROJECT` はStep 1で確認した値に置き換える）
>
> **② ウィザードの質問にすべて回答する**
>
> ウィザードから複数の質問が表示される。以下を参考に回答すること。
>
> | 質問                                                               | 回答                    | 意味                                                                    |
> | ------------------------------------------------------------------ | ----------------------- | ----------------------------------------------------------------------- |
> | Route Sentry requests through Next.js server to avoid ad blockers? | **Yes**                 | 広告ブロッカーによるSentry通信のブロックを回避する                      |
> | Enable Tracing?                                                    | **Yes**                 | ページ読み込みやAPI呼び出しの速度を計測する                             |
> | Enable Session Replay?                                             | **Yes**                 | ユーザー操作を録画してエラー時の再現に使う                              |
> | Send application logs to Sentry?                                   | **Yes**                 | console.logなどのログもSentryに送り、エラー前後の文脈がわかるようにする |
> | Create an example page to test your Sentry setup?                  | **Yes**                 | テスト用ページ（`/sentry-example-page`）を自動生成する                  |
> | Are you using a CI/CD tool?                                        | **Yes**                 | Source Mapアップロード用の認証トークンを発行する                        |
> | Did you configure CI as shown above?                               | **I'll do it later...** | Vercelへの環境変数追加は後のStepでまとめて行う                          |
> | Add a project-scoped MCP server configuration?                     | **No**                  | SentryのAIデバッグ機能。今は不要                                        |
>
> **③ ウィザードの完了を確認する**
>
> 「Successfully installed the Sentry Next.js SDK!」と表示されれば成功。
>
> 完了したら教えてください。

### ウィザードが自動生成するファイル

- `sentry.client.config.ts` — クライアントサイドのSentry初期化
- `sentry.server.config.ts` — サーバーサイドのSentry初期化
- `sentry.edge.config.ts` — Edge RuntimeのSentry初期化
- `instrumentation.ts` / `instrumentation-client.ts` — Next.jsのinstrumentation設定
- `next.config.ts` の修正（`withSentryConfig` でラップ）
- `app/global-error.tsx` — グローバルエラーハンドラー
- `app/sentry-example-page/page.tsx` — テスト用ページ
- `app/api/sentry-example-api/route.ts` — テスト用APIルート
- `.env.sentry-build-plugin` — Source Mapアップロード用の認証トークン（`.gitignore`に自動追加される）

### 完了確認

- 「Successfully installed the Sentry Next.js SDK!」と表示されていること
- 上記のファイルが生成されていること

---

## Step 3 `[USER]`：Sentryの動作確認をする

### やること

ウィザードが自動生成したテスト用ページでエラーを発生させ、Sentryダッシュボードにエラーが表示されることを確認する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. ローカル開発サーバー（`npm run dev`）を起動する
> 2. ブラウザで `http://localhost:3000/sentry-example-page` にアクセスする
> 3. ページ内のボタンをクリックしてテストエラーを送信する
> 4. Sentryダッシュボード（[sentry.io](https://sentry.io)）の **「Issues」** を開く
> 5. 以下のテストエラーが表示されていることを確認する
>    - **SentryExampleFrontendError** — クライアントサイドのテストエラー
>    - **SentryExampleAPIError** — サーバーサイドのテストエラー
>
> ⚠️ エラーがダッシュボードに反映されるまで数分かかる場合がある。
>
> 完了したら教えてください。

### 完了確認

- Sentryダッシュボードにクライアント・サーバー両方のテストエラーが表示されていること

---

## Step 4 `[AGENT]`：テスト用ページを削除する

### やること

Sentryの動作確認が完了したら、テスト用のページとAPIルートを削除する。本番環境に不要なコードを残さないため。

### 削除対象

- `app/sentry-example-page/` ディレクトリ
- `app/api/sentry-example-api/` ディレクトリ

### エージェントへの指示例

> Sentryの動作確認が完了したので、テスト用ファイルを削除してください。`app/sentry-example-page/` と `app/api/sentry-example-api/` を削除します。

### 確認ポイント

- テスト用ファイルが削除されていること
- `npm run build` がエラーなく成功すること

---

## Step 5 `[USER]`：PostHogアカウントを作成してプロジェクトキーを取得する

### やること

PostHogにアカウントを作成し、プロジェクトのAPIキーを取得する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. [posthog.com](https://posthog.com) にアクセスしてアカウントを作成する（GitHubログイン推奨）
> 2. プロジェクトを作成する（プロジェクト名は任意、例：`folio`）
> 3. 左サイドバーの「Settings」→「Project」→「General」を開く
> 4. **「Project token & ID」** セクションから以下を確認する
>    - **Project token：** `phc_` で始まるキー
>    - **Region：** `US Cloud` または `EU Cloud`
> 5. 同じページの **「SDK setup」** セクションで「JavaScript SDK」タブを選ぶと、`api_host` の値が確認できる
>    - US Cloudの場合：`https://us.i.posthog.com`
>    - EU Cloudの場合：`https://eu.i.posthog.com`
> 6. `.env.local` に以下を追加する
>
> | キー                       | 値                                             |
> | -------------------------- | ---------------------------------------------- |
> | `NEXT_PUBLIC_POSTHOG_KEY`  | Project token（`phc_...`）                     |
> | `NEXT_PUBLIC_POSTHOG_HOST` | api_hostの値（例：`https://us.i.posthog.com`） |
>
> ⚠️ PostHogのキーは公開キーのため `NEXT_PUBLIC_` プレフィックスで問題ない。
>
> 完了したら教えてください。

### 完了確認

- PostHogにプロジェクトが作成されていること
- `.env.local` に `NEXT_PUBLIC_POSTHOG_KEY`・`NEXT_PUBLIC_POSTHOG_HOST` が追加されていること

---

## Step 7 `[AGENT]`：PostHog SDKをセットアップする

### やること

PostHogのJavaScript SDKをインストールし、Next.js App Routerに統合する。

### インストールするパッケージ

- `posthog-js` — クライアントサイドのPostHog SDK

### 実装するファイル

**`lib/posthog.ts`**

- PostHogクライアントの初期化ヘルパー
- `NEXT_PUBLIC_POSTHOG_KEY` と `NEXT_PUBLIC_POSTHOG_HOST` を使用

**`components/posthog-provider.tsx`**（Client Component）

- `"use client"` を先頭に記述
- `posthog-js/react` の `PostHogProvider` を使ってアプリをラップする
- `usePathname()` と `useSearchParams()` を使ってページビューを自動キャプチャする
- PostHogの初期化は `typeof window !== 'undefined'` かつ環境変数が設定されている場合のみ行う

**`app/layout.tsx`**

- `PostHogProvider` でアプリ全体をラップする

### エージェントへの指示例

> `posthog-js` をインストールし、`components/posthog-provider.tsx` を作成してください。`"use client"` のClient Componentとして、`posthog-js/react` の `PostHogProvider` でchildrenをラップします。`usePathname()` の変更を `useEffect` で監視して `posthog.capture('$pageview')` を送信します。最後に `app/layout.tsx` を更新して `PostHogProvider` でアプリ全体をラップしてください。

### 確認ポイント

- `package.json` に `posthog-js` が含まれていること
- `components/posthog-provider.tsx` が作成されていること
- `app/layout.tsx` が `PostHogProvider` でラップされていること
- `npm run build` がエラーなく成功すること

---

## Step 8 `[USER]`：PostHogダッシュボードでページビューを確認する

### やること

ローカル環境でアプリを操作し、PostHogダッシュボードにページビューイベントが記録されることを確認する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. ローカル開発サーバー（`npm run dev`）を起動する
> 2. ブラウザでアプリのいくつかのページにアクセスする
>    - トップページ（`/`）
>    - 作品詳細ページ（`/works/[id]`）
>    - 管理ダッシュボード（`/admin`）
> 3. PostHogダッシュボード（[posthog.com](https://posthog.com)）を開く
> 4. 「Activity」または「Events」ページで `$pageview` イベントが記録されていることを確認する
>
> ⚠️ イベントが反映されるまで数分かかる場合がある。
>
> 完了したら教えてください。

### 完了確認

- PostHogダッシュボードにページビューイベントが記録されていること

---

## Step 9 `[USER]`：Lighthouseでパフォーマンスを計測する

### やること

Chrome DevToolsのLighthouseを使って、本番URL（またはローカル）のパフォーマンスを計測する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. Chromeでアプリのトップページを開く（本番URLまたは `http://localhost:3000`）
> 2. DevToolsを開く（`F12` または `Cmd + Option + I`）
> 3. 「Lighthouse」タブを選択する
> 4. 以下の設定で計測する
>    - Categories：すべてチェック（Performance / Accessibility / Best Practices / SEO）
>    - Device：Mobile
> 5. 「Analyze page load」をクリック
> 6. 計測結果のスコアを確認する
>
> **目標スコアの目安：**
>
> | カテゴリ       | 目標   | 備考                                     |
> | -------------- | ------ | ---------------------------------------- |
> | Performance    | 80以上 | 画像の最適化やフォント読み込みで改善可能 |
> | Accessibility  | 90以上 | alt属性やコントラスト比を確認            |
> | Best Practices | 90以上 | HTTPS・画像フォーマットなど              |
> | SEO            | 90以上 | メタタグ・構造化データの有無             |
>
> ⚠️ スコアが低い項目があっても、このPhaseでは「計測して現状を把握する」ことが目的。改善は必要に応じて後から行う。
>
> 完了したら教えてください。

### 完了確認

- Lighthouseの計測結果が取得できていること
- 各カテゴリのスコアを把握していること

---

## Step 10 `[USER]`：環境変数をVercelに追加してfeatureブランチをpushする

### やること

SentryとPostHogの環境変数をVercelに追加し、実装コードをデプロイする。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> **① Vercelに環境変数を追加する**
>
> Vercelダッシュボード →「Settings」→「Environment Variables」を開き、以下を追加する
>
> | キー                       | 値                                                              |
> | -------------------------- | --------------------------------------------------------------- |
> | `SENTRY_AUTH_TOKEN`        | `.env.sentry-build-plugin` に保存されたトークン（`sntrys_...`） |
> | `NEXT_PUBLIC_POSTHOG_KEY`  | PostHogのAPIキー（`phc_...`）                                   |
> | `NEXT_PUBLIC_POSTHOG_HOST` | PostHogのホストURL                                              |
>
> ⚠️ Sentryのウィザードは DSN・org・project を設定ファイルに直接書き込むため、Vercelの環境変数に追加する必要があるのは `SENTRY_AUTH_TOKEN`（Source Mapアップロード用）のみ。
>
> **② featureブランチにpushしてPRをマージする**
>
> ```bash
> git add .
> git commit -m "feat: Sentry・PostHogを追加"
> git push origin feature/monitoring
> ```
>
> GitHubでPRを作成してmainにマージする。
> VercelダッシュボードでデプロイがSuccessになるまで待つ。
>
> **③ 本番URLで動作確認する**
>
> - 本番URLにアクセスし、いくつかのページを閲覧する
> - PostHogダッシュボードにイベントが記録されることを確認する
> - （オプション）ブラウザのコンソールでエラーを発生させ、Sentryに記録されることを確認する
>
> 完了したら教えてください。

### 完了確認

- Vercelの環境変数に `SENTRY_AUTH_TOKEN`・PostHog関連の値が追加されていること
- Vercelのデプロイが成功していること
- 本番URLでPostHogにイベントが記録されること

---

## Phase 7 完了チェックリスト

- [ ] Sentryアカウントが作成されていること
- [ ] ウィザード（`npx @sentry/wizard`）が正常に完了していること
- [ ] ウィザードが生成した設定ファイル（`sentry.server.config.ts` 等）が存在すること
- [ ] Sentryダッシュボードにテストエラーが記録されたこと（テスト後に削除済み）
- [ ] PostHogアカウントが作成されていること
- [ ] `.env.local` に PostHog の環境変数が追加されていること
- [ ] `posthog-js` がインストールされ、`PostHogProvider` でアプリがラップされていること
- [ ] PostHogダッシュボードにページビューイベントが記録されること
- [ ] Lighthouseで計測を実施し、各カテゴリのスコアを把握していること
- [ ] Vercelの環境変数に `SENTRY_AUTH_TOKEN`・PostHog関連の値が追加されていること
- [ ] 本番URL（Vercel）でもSentry・PostHogが動作すること
- [ ] featureブランチで開発してPR→mainにマージした

---

## よくある失敗パターン

**Sentryにエラーが送信されない**
**Sentryにエラーが送信されない**
ウィザードが設定ファイル（`instrumentation-client.ts` や `sentry.server.config.ts`）にDSNを直接書き込んでいるか確認すること。ウィザードは `.env.local` に環境変数を追加するのではなく、設定ファイルにDSNをハードコードする。DSNの形式は `https://xxxxx@o12345.ingest.us.sentry.io/67890` のようになる。

**PostHogでページビューが記録されない**
`PostHogProvider` が `app/layout.tsx` でラップされていない、または `NEXT_PUBLIC_POSTHOG_KEY` が設定されていない。また、`usePathname()` の変更を監視する `useEffect` が正しく実装されているか確認すること。

**`npm run build` でSentry関連のエラーが出る**
**`npm run build` でSentry関連のエラーが出る**
ウィザードが `.env.sentry-build-plugin` に `SENTRY_AUTH_TOKEN` を保存している。このファイルが存在し、トークンが正しいか確認すること。Vercelデプロイ時には `SENTRY_AUTH_TOKEN` をVercelの環境変数に追加する必要がある。

**Lighthouseのスコアが極端に低い**
開発モード（`npm run dev`）で計測するとスコアが低くなる。正確なスコアを計測するには `npm run build && npm start` でプロダクションビルドを起動するか、本番URL（Vercel）で計測すること。

**PostHogの初期化でサーバーサイドエラーが出る**
PostHogはクライアントサイドでのみ動作する。`typeof window !== 'undefined'` のチェックを入れること。また、`PostHogProvider` は `"use client"` を必ずつけること。

---

## 次のPhase

Phase 7が完了したら、Phase 8（環境変数管理）に進む。  
Phase 8では `.env.local` の全変数がVercelに設定されているかを確認し、READMEに必要な環境変数一覧を記載する。

---

_このガイドはFolio研修カリキュラムの一部です。不明点はメンターに確認してください。_
