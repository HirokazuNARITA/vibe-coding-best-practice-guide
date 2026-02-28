# Phase 2 手順書：認証のセットアップ

> **ゴール：** `/admin` にログインしないとアクセスできない状態にする  
> **所要時間の目安：** 2〜3時間  
> **前提：** Phase 1 が完了していること

---

## エージェント向け指示

このドキュメントの各Stepには実施者ラベルが付いています。

- **`[USER]`** — ユーザーが手動で実施する作業（ダッシュボード操作・アカウント作成など）。エージェントはこのStepの「ユーザーへの提示内容」をユーザーにそのまま提示し、完了確認を待ってから次のStepに進むこと。
- **`[AGENT]`** — エージェントが実施する作業（コード実装・コマンド実行など）。エージェントが自律的に実行すること。

---

## このPhaseで学ぶこと

- **Supabase Auth** を使ったメール・パスワード認証
- **Next.js middleware** を使ったページ保護
- **セッション管理** の仕組み（クライアント・サーバーそれぞれでのセッション取得）
- **認証は絶対に自作しない** という原則の体験

---

## このPhaseで使うツール

| ツール        | 役割                          | 料金                        |
| ------------- | ----------------------------- | --------------------------- |
| Supabase Auth | 認証基盤                      | 無料枠あり（MAU 5万人まで） |
| @supabase/ssr | Next.js App RouterでのSSR対応 | 無料（OSS）                 |

---

## このPhaseで実装すること

- [ ] Supabase Auth でオーナーアカウントを作成
- [ ] ログイン画面 `/admin/login` の実装
- [ ] `middleware.ts` で `/admin` 以下を保護（未認証ならログインにリダイレクト）
- [ ] ログアウト機能

---

## Step 1 `[USER]`：オーナーアカウントを作成する

### やること

Supabaseダッシュボードから管理者（オーナー）のアカウントを手動で作成する。

### なぜコードで作らないのか

このアプリのオーナーは自分1人だけ。サインアップ機能を実装すると誰でもアカウントを作れてしまう。ダッシュボードから直接作成することで、余計な機能を実装せずに済む。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. Supabaseダッシュボード →「Authentication」→「Users」を開く
> 2. 「Add user」→「Create new user」をクリック
> 3. メールアドレスとパスワードを入力して作成
> 4. 作成したメールアドレスとパスワードをメモしておく（`.env.local` には入れない）
>
> 完了したら教えてください。

### 完了確認

- Usersにアカウントが表示されていること

---

## Step 2 `[AGENT]`：Supabase SSRパッケージをインストールする

### やること

Next.js App RouterでSupabase Authを正しく使うためのパッケージ `@supabase/ssr` をインストールする。Phase 1でインストール済みの場合はスキップ。

### なぜ `@supabase/ssr` が必要なのか

Next.js App RouterはServer ComponentsとClient Componentsが混在する。それぞれの環境でセッションを正しく取得・更新するには `@supabase/ssr` が必要。

### 確認ポイント

- `package.json` に `@supabase/ssr` が含まれていること

---

## Step 3 `[AGENT]`：Supabaseクライアントを整備する

### やること

Server Components用・Client Components用・middleware用の3種類のSupabaseクライアントを `lib/` に用意する。

### なぜ3種類必要なのか

| 用途                | ファイル                     | 理由                                                 |
| ------------------- | ---------------------------- | ---------------------------------------------------- |
| Client Components用 | `lib/supabase.ts`            | Phase 1で作成済み。変更不要                          |
| Server Components用 | `lib/supabase-server.ts`     | サーバー側でCookieを読み取りセッションを取得するため |
| middleware用        | `lib/supabase-middleware.ts` | リクエスト単位でセッションを検証・更新するため       |

### エージェントへの指示例

> `@supabase/ssr` を使って、Server Components用・middleware用のSupabaseクライアントを作成してください。Supabaseの公式ドキュメントのNext.js App Router向けの実装に従ってください。

### 確認ポイント

- `lib/` に3種類のクライアントファイルが存在すること

---

## Step 4 `[AGENT]`：middleware.ts を実装する

### やること

`/admin` 以下へのアクセスを認証で保護する。未認証ユーザーは `/admin/login` にリダイレクトされる。

### ポイント

- `middleware.ts` はプロジェクトルートに配置する
- `matcher` の設定で `/admin` 以下のみに適用させる
- `/admin/login` 自体は除外しないと無限リダイレクトになるため注意

### 確認ポイント

- ブラウザで `/admin` にアクセスすると `/admin/login` にリダイレクトされること

---

## Step 5 `[AGENT]`：ログイン画面を実装する

### やること

`/admin/login` にメール・パスワードのログインフォームを作成する。

### 要件

- shadcn/ui のコンポーネントを使ったフォーム
- 認証エラー時にエラーメッセージを表示する
- ログイン成功後に `/admin` にリダイレクトする
- ログインはClient Componentで実装する（`supabase.auth.signInWithPassword` を使用）

### 確認ポイント

- `/admin/login` にアクセスするとログインフォームが表示されること
- 正しい認証情報でログインすると `/admin` にリダイレクトされること
- 間違った認証情報ではエラーメッセージが表示されること

---

## Step 6 `[AGENT]`：ログアウト機能を実装する

### やること

管理画面にログアウトボタンを追加する。

### 要件

- `components/logout-button.tsx` としてClient Componentで作成する
- ログアウト後は `/admin/login` にリダイレクトする
- `app/admin/page.tsx` にログアウトボタンを配置する
- `app/admin/page.tsx` はServer Componentとして実装し、サーバー側でセッションを確認する

### 確認ポイント

- ログアウトボタンをクリックすると `/admin/login` にリダイレクトされること
- ログアウト後に `/admin` にアクセスすると `/admin/login` にリダイレクトされること

---

## Phase 2 完了チェックリスト

- [ ] Supabaseダッシュボードにオーナーアカウントが存在する
- [ ] `/admin` にアクセスすると `/admin/login` にリダイレクトされる
- [ ] 正しい認証情報でログインできる
- [ ] ログイン後に `/admin` ダッシュボードが表示される
- [ ] ログアウトボタンで `/admin/login` にリダイレクトされること
- [ ] featureブランチで開発してPR→mainにマージした

---

## よくある失敗パターン

**ログイン後に画面が更新されない**
ログイン成功後に `router.refresh()` を呼ぶ必要がある。Next.jsのキャッシュをクリアするために必要。エージェントに伝えること。

**`/admin/login` にアクセスすると無限リダイレクトになる**
middleware が `/admin/login` 自体も保護対象にしてしまっている。`/admin/login` はmiddlewareの対象から除外する必要がある。

**Server Componentでセッションが取得できない**
Client用のクライアント（`lib/supabase.ts`）をServer Componentで使っている。Server Componentでは `lib/supabase-server.ts` を使うこと。

---

## 次のPhase

Phase 2が完了したら、Phase 3（UI構築）に進む。  
Phase 3では shadcn/ui を使ってトップページ・作品詳細ページ・管理ダッシュボードのUIを実装する。

---

_このガイドはFolio研修カリキュラムの一部です。不明点はメンターに確認してください。_
