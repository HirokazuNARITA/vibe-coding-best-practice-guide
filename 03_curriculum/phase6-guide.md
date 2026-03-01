# Phase 6 手順書：決済

> **ゴール：** 有料作品をStripeで購入できる状態にする  
> **所要時間の目安：** 2〜3時間  
> **前提：** Phase 5 が完了していること

---

## エージェント向け指示

このドキュメントの各Stepには実施者ラベルが付いています。

- **`[USER]`** — ユーザーが手動で実施する作業（ダッシュボード操作・アカウント作成など）。エージェントはこのStepの「ユーザーへの提示内容」をユーザーにそのまま提示し、完了確認を待ってから次のStepに進むこと。
- **`[AGENT]`** — エージェントが実施する作業（コード実装・コマンド実行など）。エージェントが自律的に実行すること。

---

## このPhaseで学ぶこと

- **Stripe Checkout** を使った決済フローの仕組み
- **決済システムを自作しない** という原則の体験
- **Webhook** の基本概念（外部サービスからの非同期通知）
- **テストモードと本番モード** の切り替え方

---

## このPhaseで使うツール

| ツール     | 役割                      | 料金                       |
| ---------- | ------------------------- | -------------------------- |
| Stripe     | 決済処理                  | 手数料のみ（売上の3.6%〜） |
| Stripe CLI | ローカルでWebhookをテスト | 無料                       |

---

## このPhaseで実装すること

- [ ] Stripeアカウント作成・APIキー取得
- [ ] Stripeクライアントのセットアップ（`lib/stripe.ts`）
- [ ] Checkout セッション作成（`actions/stripe.ts`）
- [ ] 作品詳細ページに「購入する」ボタンを追加
- [ ] 購入完了ページ（`/works/[id]/success`）の実装
- [ ] Webhook エンドポイントの実装（最小限）
- [ ] Vercelの環境変数に追加

---

## Stripe Checkout とは

決済UIをまるごと引き受けてくれるStripeのホスト型決済ページ。自前でやろうとすると以下をすべて実装・管理する必要があるが、Stripe Checkoutはそれをすべて肩代わりする。

| 自前でやる場合                          | Stripe Checkoutが解決してくれること        |
| --------------------------------------- | ------------------------------------------ |
| クレジットカード入力フォームを作る      | Stripeが管理する安全なページに遷移するだけ |
| PCI DSS準拠（カード情報の安全な取扱い） | Stripe側で対応済み                         |
| 各種カード・決済手段への対応            | 設定だけで対応できる                       |
| 不正検知・セキュリティ対応              | Stripe側で管理されている                   |

**購入フローのイメージ：**

```
訪問者が「購入する」ボタンをクリック
  ↓
Server ActionでStripe CheckoutセッションをAPIで作成
  ↓
StripeのホストされたURLにリダイレクト
  ↓
訪問者がStripeのページでカード情報を入力・支払い
  ↓
成功すると `/works/[id]/success` にリダイレクト
  ↓
StripeがWebhookで購入完了をサーバーに通知
```

---

## サンドボックスについて

Stripeはアカウント作成直後から**サンドボックス（テスト環境）と本番環境が完全に分離**されている。サンドボックスでは実際の決済は一切発生しない。

> 💡 以前は「テストモード」と呼ばれていたが、現在は「サンドボックス」という名称に変わっている。機能は同じ。

**このPhaseはすべてサンドボックスで進める。** 本番環境への切り替えはPhase完了後、実際に販売を開始するタイミングで行う。

### サンドボックスと本番環境の違い

|                         | サンドボックス                             | 本番環境                     |
| ----------------------- | ------------------------------------------ | ---------------------------- |
| APIキーのプレフィックス | `pk_test_` / `sk_test_`                    | `pk_live_` / `sk_live_`      |
| 実際の決済              | 発生しない                                 | 発生する                     |
| テスト用カード番号      | 使える                                     | 使えない                     |
| 本人確認                | 不要                                       | 必要（法人確認・身分証明書） |
| ダッシュボードの見た目  | 上部に「サンドボックス」バナーが表示される | バナーなし                   |

### テスト用カード番号

Stripeが用意したカード番号を使うことで、実際のカードなしに決済フローを動かせる。

| カード番号            | 結果                 |
| --------------------- | -------------------- |
| `4242 4242 4242 4242` | 支払い成功           |
| `4000 0000 0000 0002` | カード拒否           |
| `4000 0025 0000 3155` | 3Dセキュア認証が必要 |

有効期限は**未来の任意の日付**、CVCは**任意の3桁**でよい。

### 本番環境への切り替え方

販売を開始する準備ができたら以下の手順で切り替える。

1. Stripeダッシュボード右上の「本番環境のアカウントに切り替える」をクリック
2. 本番用の本人確認（法人確認・身分証明書など）を完了する
3. 本番用のAPIキー（`pk_live_` / `sk_live_`）を取得する
4. Vercelの環境変数を本番用キーに差し替えて再デプロイする
5. 本番用WebhookエンドポイントをStripeに再登録する

> ⚠️ サンドボックスと本番環境はAPIキーが別物。サンドボックス中に使っていた `pk_test_` / `sk_test_` キーをそのまま本番に使うことはできない。

---

## Step 1 `[USER]`：Stripeアカウントを作成してAPIキーを取得する

### やること

Stripeにアカウントを作成し、テスト用のAPIキーを取得する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. [stripe.com](https://stripe.com) にアクセスしてアカウントを作成する
> 2. 「サンドボックスを試す」画面が表示されたら **「サンドボックスに移動」** をクリックする（「本番アカウントを今すぐ取得する」は押さないこと）
> 3. ダッシュボード上部に **「サンドボックス」バナー** が表示されていることを確認する
> 4. ホーム画面右側の「APIキー」から以下の2つを取得する
>
> | キー                              | 環境変数名                           |
> | --------------------------------- | ------------------------------------ |
> | 公開可能キー（`pk_test_...`）     | `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` |
> | シークレットキー（`sk_test_...`） | `STRIPE_SECRET_KEY`                  |
>
> 5. `.env.local` に追加する
> 6. `NEXT_PUBLIC_APP_URL` も `.env.local` に追加する（ローカル開発時は `http://localhost:3000`）
>
> ⚠️ シークレットキー（`sk_test_...`）は絶対にGitHubに公開しないこと。
> ⚠️ `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` は `pk_test_...` で始まるキー。`sk_test_...` には `NEXT_PUBLIC_` をつけないこと。
>
> 完了したら教えてください。

### 完了確認

- `.env.local` に `STRIPE_SECRET_KEY`・`NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`・`NEXT_PUBLIC_APP_URL` が追加されていること
- ダッシュボード上部に「サンドボックス」バナーが表示されていること

---

## Step 2 `[AGENT]`：パッケージをインストールしてStripeクライアントを実装する

### やること

StripeのNode.js SDKをインストールし、`lib/stripe.ts` にクライアントを実装する。

### インストールするパッケージ

- `stripe` — サーバーサイドのStripe SDK

### `lib/stripe.ts` の要件

- `STRIPE_SECRET_KEY` を使ってStripeクライアントを初期化する
- シングルトンパターンで実装する（複数インスタンスが生成されないようにする）

### エージェントへの指示例

> `stripe` パッケージをインストールして、`lib/stripe.ts` にStripeクライアントをシングルトンパターンで実装してください。`STRIPE_SECRET_KEY` 環境変数を使って初期化します。

### 確認ポイント

- `package.json` に `stripe` が含まれていること
- `lib/stripe.ts` が実装されていること

---

## Step 3 `[AGENT]`：Checkout セッション作成の Server Action を実装する

### やること

`actions/stripe.ts` にCheckoutセッションを作成するServer Actionを実装する。Phase 1で空ファイルとして用意済み。

### 要件

- 関数名：`createCheckoutSession`
- 引数：`workId`（作品ID）
- 処理：
  1. `workId` でDBから作品情報を取得する
  2. Stripe APIでCheckoutセッションを作成する
  3. セッションのURLにリダイレクトする（`redirect()`を使用）
- Checkoutセッションの設定：
  - `mode`: `'payment'`
  - `line_items`: 作品のタイトル・価格・数量1
  - `success_url`: `${NEXT_PUBLIC_APP_URL}/works/[workId]/success`
  - `cancel_url`: `${NEXT_PUBLIC_APP_URL}/works/[workId]`
  - `metadata`: `{ workId }` （Webhookで使用するため）

### エージェントへの指示例

> `actions/stripe.ts` に `createCheckoutSession(workId: string)` を実装してください。DBから作品を取得してStripe Checkoutセッションを作成し、セッションURLにリダイレクトします。success_urlは `/works/[workId]/success`、cancel_urlは `/works/[workId]` にしてください。metadataにworkIdを含めてください。

### 確認ポイント

- `actions/stripe.ts` に `createCheckoutSession` が実装されていること

---

## Step 4 `[AGENT]`：作品詳細ページに「購入する」ボタンを追加する

### やること

Phase 3で実装した作品詳細ページ（`app/(public)/works/[id]/page.tsx`）に「購入する」ボタンを追加し、`createCheckoutSession` を呼び出す。

### 要件

- 有料作品（`price > 0`）のみ「購入する」ボタンを表示する
- 無料作品（`price === 0`）は「無料で見る」ボタンを表示する（このPhaseではリンク先は `#` でよい）
- ボタンはClient Componentとして実装する（Server Actionの呼び出しにformを使う）
- ボタンクリックで `createCheckoutSession` を呼び出す

### エージェントへの指示例

> 作品詳細ページ（`app/(public)/works/[id]/page.tsx`）に購入ボタンを追加してください。price > 0 の有料作品には「購入する」ボタンを表示し、クリックすると `createCheckoutSession` を呼び出します。price === 0 の無料作品には「無料で見る」ボタン（リンク先は#）を表示します。ボタン部分はClient Componentとして切り出してください。

### 確認ポイント

- 有料作品の詳細ページに「購入する」ボタンが表示されること
- 無料作品の詳細ページに「無料で見る」ボタンが表示されること

---

## Step 5 `[AGENT]`：購入完了ページを実装する

### やること

`app/(public)/works/[id]/success/page.tsx` に購入完了ページを実装する。Phase 1で空ファイルとして用意済み。

### 要件

- 「ありがとうございました」メッセージを表示する
- 作品名を表示する（`workId` でDBから取得）
- トップページに戻るリンクを含む
- Server Componentで実装する

### エージェントへの指示例

> `app/(public)/works/[id]/success/page.tsx` を実装してください。paramsのidでDBから作品を取得して作品名を表示し、「ご購入ありがとうございました」メッセージとトップページへのリンクを含むServer Componentです。

### 確認ポイント

- `/works/[id]/success` にアクセスすると購入完了ページが表示されること

---

## Step 6 `[AGENT]`：Webhook エンドポイントを実装する

### やること

`app/api/stripe/webhook/route.ts` にStripeからの購入完了通知を受け取るWebhookエンドポイントを実装する。

### Webhookとは

Stripeが「支払いが完了した」などのイベントを発生させたとき、指定したURLに自動でPOSTリクエストを送ってくれる仕組み。これがないと「Stripeで支払いは完了しているのにアプリ側が気づかない」という状態になる。

### このPhaseでの実装スコープ（最小限）

- `checkout.session.completed` イベントを受け取る
- Stripeの署名（`STRIPE_WEBHOOK_SECRET`）を検証する
- コンソールにログを出力するだけでよい（DBへの記録はスコープ外）

> 💡 本来はWebhookを受け取ったらDBに購入記録を保存するが、このアプリは購入管理機能を持たないため、署名検証の実装体験のみを目的とする。

### 要件

- `app/api/stripe/webhook/route.ts` に `POST` ハンドラーを実装する
- `stripe.webhooks.constructEvent()` で署名を検証する
- `checkout.session.completed` イベント受信時にコンソールログを出力する
- 署名検証に失敗した場合は400を返す

### エージェントへの指示例

> `app/api/stripe/webhook/route.ts` にStripe Webhookエンドポイントを実装してください。POSTリクエストを受け取り、`stripe.webhooks.constructEvent()` で署名を検証します。`checkout.session.completed` イベント受信時はコンソールにログを出力するだけでよいです。署名検証失敗時は400を返してください。

### 確認ポイント

- `app/api/stripe/webhook/route.ts` が実装されていること

---

## Step 7 `[USER]`：Stripe CLI でローカルのWebhookをテストする

### やること

Stripe CLIをインストールし、ローカル開発環境でWebhookのテストを行う。あわせて `STRIPE_WEBHOOK_SECRET` を取得する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> **① Stripe CLI のインストール**
>
> Macの場合：
>
> ```bash
> brew install stripe/stripe-cli/stripe
> ```
>
> **② Stripe CLIでログイン**
>
> ```bash
> stripe login
> ```
>
> ブラウザが開くので、Stripeアカウントで認証する。
>
> **③ ローカルにWebhookをフォワードする**
>
> 開発サーバー（`npm run dev`）を起動した状態で以下を実行：
>
> ```bash
> stripe listen --forward-to localhost:3000/api/stripe/webhook
> ```
>
> 実行すると以下のような出力が表示される：
>
> ```
> Ready! Your webhook signing secret is whsec_xxxxxxxxxxxx
> ```
>
> この `whsec_...` の値を `.env.local` の `STRIPE_WEBHOOK_SECRET` に設定する。
>
> **④ 動作確認**
>
> 別のターミナルで以下を実行してテストイベントを送信する：
>
> ```bash
> stripe trigger checkout.session.completed
> ```
>
> 開発サーバーのログに `checkout.session.completed` が出力されれば成功。
>
> 完了したら教えてください。

### 完了確認

- `.env.local` に `STRIPE_WEBHOOK_SECRET`（`whsec_...`）が追加されていること
- テストイベント送信後にサーバーログにイベントが出力されること

---

## Step 8 `[USER]`：環境変数をVercelに追加する

### やること

StripeのAPIキーをVercelの環境変数に追加する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. Vercelダッシュボード →「Settings」→「Environment Variables」を開く
> 2. 以下の4つを追加する
>
> | キー                                 | 値                                                       |
> | ------------------------------------ | -------------------------------------------------------- |
> | `STRIPE_SECRET_KEY`                  | Stripeダッシュボードのシークレットキー（sk*test*...）    |
> | `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Stripeダッシュボードの公開可能キー（pk*test*...）        |
> | `STRIPE_WEBHOOK_SECRET`              | 本番用Webhookシークレット（Step 9で取得）                |
> | `NEXT_PUBLIC_APP_URL`                | VercelのデプロイURL（例：`https://my-folio.vercel.app`） |
>
> ⚠️ `STRIPE_WEBHOOK_SECRET` はStep 9（本番Webhookエンドポイント登録）で取得する値を使う。Step 7で取得したローカル用の `whsec_...` とは別の値になる。
>
> 3. 追加後に再デプロイする
>
> 完了したら教えてください。

### 完了確認

- Vercelの環境変数に4つが追加されていること

---

## Step 9 `[USER]`：本番用WebhookエンドポイントをStripeに登録する

### やること

Stripeダッシュボードに本番（Vercel）のWebhook URLを登録し、本番用の `STRIPE_WEBHOOK_SECRET` を取得する。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. Stripeダッシュボード →「開発者」→「Webhook」を開く
> 2. 「エンドポイントを追加」をクリック
> 3. エンドポイントURLに以下を入力する（YOUR_APP_URLは実際のVercel URLに置き換える）
>    ```
>    https://YOUR_APP_URL/api/stripe/webhook
>    ```
> 4. 「リッスンするイベント」で `checkout.session.completed` を選択する
> 5. 「エンドポイントを追加」で保存する
> 6. 登録したエンドポイントの詳細ページを開き、「署名シークレット」をコピーする
> 7. Vercelの環境変数 `STRIPE_WEBHOOK_SECRET` をこの値で更新する
> 8. Vercelを再デプロイする
>
> 完了したら教えてください。

### 完了確認

- Stripeダッシュボードにエンドポイントが登録されていること
- Vercelの `STRIPE_WEBHOOK_SECRET` が本番用の値に更新されていること

---

## Phase 6 完了チェックリスト

- [ ] Stripeアカウントが作成されていること（テストモード）
- [ ] `.env.local` に Stripe の環境変数が追加されていること
- [ ] 有料作品の詳細ページに「購入する」ボタンが表示されること
- [ ] 「購入する」ボタンをクリックするとStripe Checkoutに遷移すること
- [ ] テストカード番号（`4242 4242 4242 4242`）で購入が完了すること
- [ ] 購入完了後に `/works/[id]/success` にリダイレクトされること
- [ ] ローカルでWebhookのテストイベントが受信できること
- [ ] Vercelの環境変数にStripeのキーが追加されていること
- [ ] 本番WebhookエンドポイントがStripeに登録されていること
- [ ] featureブランチで開発してPR→mainにマージした

---

## よくある失敗パターン

**「購入する」ボタンを押してもStripeに遷移しない**
`createCheckoutSession` の `NEXT_PUBLIC_APP_URL` が未設定か空の場合、`success_url` が不正になりCheckoutセッション作成がエラーになる。`.env.local` に `NEXT_PUBLIC_APP_URL=http://localhost:3000` が設定されているか確認すること。

**購入完了後に `/success` に遷移しない**
`success_url` の末尾に `?session_id={CHECKOUT_SESSION_ID}` をつけ忘れた場合や、URLのパスが間違っている場合に発生する。`createCheckoutSession` の `success_url` を確認すること。

**Webhookの署名検証が失敗する（400エラー）**
ローカル開発時は `stripe listen` で取得した `whsec_...` を使うこと。本番では Stripe ダッシュボードのエンドポイント詳細ページから取得した値を使う。2つの値は別物なので混同しないこと。

**Webhookエンドポイントでリクエストボディのパースに失敗する**
Next.jsのデフォルトでは `request.json()` でボディをパースするが、Webhookの署名検証には生のバイト列（`request.arrayBuffer()` または `request.text()`）が必要。エージェントへの指示に「生のボディを使って署名検証すること」と明記すること。

---

## 次のPhase

Phase 6が完了したら、Phase 7（監視・分析）に進む。  
Phase 7ではSentryでエラー監視、PostHogでユーザー行動分析、Lighthouseでパフォーマンス計測をセットアップする。

---

_このガイドはFolio研修カリキュラムの一部です。不明点はメンターに確認してください。_
