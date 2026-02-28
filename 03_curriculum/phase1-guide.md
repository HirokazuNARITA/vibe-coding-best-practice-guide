# Phase 1 手順書：デプロイ基盤のセットアップ

> **ゴール：** 空のNext.jsアプリがVercelの公開URLで動いている状態にする
> **所要時間の目安：** 2〜3時間
> **前提知識：** ターミナルの基本操作、GitHubアカウント取得済み

---

## エージェント向け指示

このドキュメントの各Stepには実施者ラベルが付いています。

- **`[USER]`** — ユーザーが手動で実施する作業（ダッシュボード操作・アカウント作成など）。エージェントはこのStepの「ユーザーへの提示内容」をユーザーにそのまま提示し、完了確認を待ってから次のStepに進むこと。
- **`[AGENT]`** — エージェントが実施する作業（コード実装・コマンド実行など）。エージェントが自律的に実行すること。

---

## このPhaseで学ぶこと

- **push-to-deploy** の体験：コードをpushするだけで自動デプロイされる仕組み
- **環境変数** の正しい扱い方：シークレットをコードに書かない原則
- **プロジェクト構造** の設計：後から機能を追加しやすい構成にする
- **README** を最初から書く習慣

---

## このPhaseで使うツール

| ツール       | 役割                      | 料金        |
| ------------ | ------------------------- | ----------- |
| Next.js      | Webアプリのフレームワーク | 無料（OSS） |
| Tailwind CSS | スタイリング              | 無料（OSS） |
| shadcn/ui    | UIコンポーネント          | 無料（OSS） |
| Prisma       | DB操作のORM               | 無料（OSS） |
| Supabase     | Postgresデータベース      | 無料枠あり  |
| GitHub       | ソースコード管理          | 無料        |
| Vercel       | デプロイ・ホスティング    | 無料枠あり  |

> **注意：** このPhaseではDBへの書き込みや読み込みは行わない。Prismaの接続設定まで行い、実際のDB操作はPhase 4で行う。

---

## 事前準備 [USER]：アカウント作成

### やること

作業を始める前に以下のアカウントをすべて作成しておくこと。すべて無料で始められる。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> - [ ] [GitHub](https://github.com) — ソースコード管理
> - [ ] [Supabase](https://supabase.com) — GitHubアカウントでログイン推奨
> - [ ] [Vercel](https://vercel.com) — GitHubアカウントでログイン推奨
>
> 完了したら教えてください。

### 完了確認

- 3つのアカウントがすべて作成されていること

---

## Step 1 `[AGENT]`：Next.jsプロジェクトを作成する

### やること

ターミナルで `create-next-app` コマンドを実行し、プロジェクトを作成する。

### 選択肢の意味

コマンド実行中にいくつか質問が出る。以下の意図で選択すること。

| 質問             | 選択 | 理由                                   |
| ---------------- | ---- | -------------------------------------- |
| TypeScript       | Yes  | 型安全なコードでバグを早期発見するため |
| ESLint           | Yes  | コードの品質を自動チェックするため     |
| Tailwind CSS     | Yes  | このプロジェクトのスタイリング方法     |
| `src/` directory | No   | App Routerではsrcなしが一般的          |
| App Router       | Yes  | Next.js 13以降の推奨方式               |
| Turbopack        | No   | まだ安定版ではないため                 |
| Import alias     | No   | デフォルトの`@/`で十分                 |

### 確認ポイント

- `npm run dev` でローカルサーバーが起動し、`http://localhost:3000` が開けること
- ブラウザにNext.jsのデフォルト画面が表示されること

---

## Step 2 `[AGENT]`：shadcn/ui をセットアップする

### やること

shadcn/ui の初期化コマンドを実行し、最低限必要なコンポーネントを追加する。

### shadcn/ui とは

コピペで使えるUIコンポーネント集。`Button` や `Card` などを1コマンドで追加できる。自分でCSSを書く必要がなく、デザインの一貫性が保たれる。

> **重要：** shadcn/ui は「インストール」ではなく「コードをプロジェクトに追加する」という仕組み。追加したコンポーネントは `components/ui/` フォルダに入り、自由にカスタマイズできる。

### このPhaseで追加するコンポーネント

- `button`
- `card`
- `input`
- `badge`

### 確認ポイント

- `components/ui/` フォルダが作成されていること
- 各コンポーネントのファイルが存在すること

---

## Step 3 `[AGENT]`：Prisma をセットアップする

### やること

PrismaをインストールしてSupabaseのDBと接続する設定を行う。

### Prisma とは

DBのスキーマ（テーブル設計）をコードで管理し、型安全なDB操作を可能にするORM。生のSQLを書く代わりに、TypeScriptで直感的にDB操作ができる。

### なぜ生のSQLを書かないのか

- 書き方のミスでSQLインジェクション脆弱性が発生しやすい
- テーブル構造の変更管理（マイグレーション）が手動になる
- TypeScriptの型補完が効かない

### このPhaseでやること・やらないこと

- ✅ Prismaのインストールと初期設定
- ✅ `schema.prisma` にデータモデルを定義
- ✅ Supabaseへの接続設定（`DATABASE_URL`）
- ❌ 実際のマイグレーション実行（Phase 4で行う）
- ❌ DBへの読み書き（Phase 4で行う）

### 確認ポイント

- `prisma/schema.prisma` ファイルが存在すること
- `lib/prisma.ts` にPrismaクライアントが定義されていること

---

## Step 4 `[USER]`：Supabase プロジェクトを作成する

### やること

Supabaseで新しいプロジェクトを作成し、接続情報を取得する。

### Supabase とは

PostgreSQLのデータベースをクラウドで管理してくれるサービス。サーバーの設定・管理が不要で、接続情報を取得するだけで使い始められる。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> **1. プロジェクト作成時の設定**
>
> - **Region（リージョン）：** Northeast Asia (Tokyo) を選択（ユーザーに近いリージョンを選ぶとレスポンスが速くなるため）
> - **Database Password：** 安全なパスワードを設定し、必ずメモしておくこと
> - **Enable Data API：** ON（チェックを入れる）— SupabaseのクライアントライブラリやAPIを通じてDBにアクセスするための設定。このアプリではSupabase Authを使うため必須
> - **Enable automatic RLS：** ON（チェックを入れる）— RLSとはRow Level Security（行レベルセキュリティ）の略。automatic RLSをONにすると、マイグレーションで作られたテーブルにも自動でRLSが有効になる。必ずONにすること
>
> > ⚠️ RLSを有効にしただけではまだアクセス制御のルール（ポリシー）は定義されていない。ポリシーの設定はPhase 2以降で行う。
>
> **2. 取得する情報**
>
> - **Project URL：** 「Integrations → Data API」を開き、「API URL」をコピーする
> - **Publishable key / Secret key：** 「Project Settings → API Keys」を開き、以下を取得する
>
> | 項目            | 用途                                          |
> | --------------- | --------------------------------------------- |
> | Publishable key | 公開して良いAPIキー（フロントエンドから使用） |
> | Secret key      | 秘密のAPIキー（サーバーサイドのみで使用）     |
>
> - **Connection string (URI)：** 画面上部の「**Connect**」ボタンから **2種類** 取得する
>
> | 用途                             | Method            | 取得手順                                                                                               |
> | -------------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------ |
> | **ローカル開発**（`.env.local`） | Direct connection | 「Connection String」タブ → Type: URI、Method: **Direct connection** を選択 → 表示された文字列をコピー |
> | **Vercel デプロイ**（環境変数）  | Transaction       | 「Connection String」タブ → Type: URI、Method: **Transaction** を選択 → 表示された文字列をコピー       |
>
> > ⚠️ **重要：** Direct connection（`db.xxx.supabase.co:5432`）は Vercel のサーバーレス環境から接続できない。Vercel には必ず **Transaction** モード（`aws-0-xxx.pooler.supabase.com:6543`）の接続文字列を使うこと。
>
> > ⚠️ 文字列内の `[PASSWORD]` の部分はプロジェクト作成時に設定したDBパスワードに置き換えること。
>
> > ⚠️ `Secret key` と `Connection string` は絶対にGitHubに公開しないこと。
>
> 完了したら教えてください。

### 完了確認

- Supabaseのダッシュボードでプロジェクトが「Active」になっていること
- Direct connection と Transaction の **2種類** の接続文字列をメモしてあること

---

## Step 5 `[USER]`：環境変数を設定する

### やること

プロジェクトのルートに `.env.local` ファイルを作成し、取得した接続情報を記入する。

### 環境変数とは

APIキーやパスワードなどの「秘密の値」をコードに直接書かず、外部から注入する仕組み。

### なぜコードに直接書いてはいけないのか

コードはGitHubで管理する。もしAPIキーをコードに書いてしまうと、リポジトリを見た人全員にキーが漏れる。GitHubは公開リポジトリに含まれるシークレットを自動検出し、サービス側にも通知する仕組みがある。一度漏れたキーは無効化して再発行するしかない。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. プロジェクトのルートに `.env.local` ファイルを作成する
> 2. 取得した接続情報を記入する
>
> **設定する値**
>
> - Project URL、Publishable key、Secret key：そのまま
> - **DATABASE_URL：** ローカル開発用なので **Direct connection** の接続文字列を使う
>
> > 💡 Transaction の接続文字列は Step 8 で Vercel に設定するときに使う。`.env.local` には入れない。
>
> **`NEXT_PUBLIC_` プレフィックスのルール**
>
> | プレフィックス      | ブラウザに公開 | 用途                         |
> | ------------------- | -------------- | ---------------------------- |
> | `NEXT_PUBLIC_` あり | される         | フロントエンドから使うもの   |
> | `NEXT_PUBLIC_` なし | されない       | サーバーサイドのみで使うもの |
>
> `service_role key` や `DATABASE_URL` は絶対に `NEXT_PUBLIC_` をつけないこと。
>
> 3. `.gitignore` に `.env*` または `.env.local` が含まれているか確認する（`create-next-app` で作成したプロジェクトには最初から含まれている）
>
> 完了したら教えてください。

### 完了確認

- `.env.local` ファイルが存在し、全変数が記入されていること
- `.gitignore` に `.env*` または `.env.local` が含まれていること

---

## Step 6 `[AGENT]`：フォルダ構造を整える

### やること

仕様書のフォルダ構造に従って、必要なフォルダとファイルを作成する。中身は空でよい。

### なぜ最初に構造を決めるのか

後から構造を変えると、インポートパスの修正など余計な作業が増える。最初に決めておくことで、機能追加のたびに「どこに書くか」で迷わなくなる。

### フォルダの役割

| フォルダ      | 役割                                    |
| ------------- | --------------------------------------- |
| `app/`        | ページとAPIルート（Next.js App Router） |
| `components/` | 再利用するUIコンポーネント              |
| `lib/`        | 外部サービスのクライアント設定          |
| `actions/`    | Server Actions（DBやAPIへの操作）       |
| `types/`      | TypeScriptの型定義                      |
| `prisma/`     | DBスキーマとマイグレーション            |

### 確認ポイント

- 仕様書のフォルダ構造と一致していること
- `lib/supabase.ts`、`lib/prisma.ts` が存在すること（中身はクライアント初期化のみでよい）

---

## Step 7 `[USER]`：GitHubにpushする

### やること

GitHubでリポジトリを作成し、コードをpushする。

### リポジトリの公開設定

**Publicを推奨する。** 理由は以下の通り。

- VercelはPublicリポジトリとの連携が無制限で使える
- 研修用アプリなので機密情報はない
- APIキーは `.env.local` で管理するためコードに含まれない

Publicにする場合に必ず守ること：

- `.env.local` を `.gitignore` に含めてからpushする（Next.jsはデフォルトで含まれている）
- APIキーをコードにハードコードしない

この2点を守れば、Publicでも安全に運用できる。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> 1. GitHubで新しいリポジトリを作成する（Public推奨）
> 2. ローカルで `git remote add origin` と `git push` を実行してコードをpushする
>
> **ブランチ運用のルール**
>
> | ブランチ      | 用途                                 |
> | ------------- | ------------------------------------ |
> | `main`        | 本番環境と連動。直接pushしない       |
> | `feature/xxx` | 機能開発用。PRを経由してmainにマージ |
>
> > **重要：** `main` に直接pushしない。1回の誤ったpushが本番環境を壊す可能性がある。
>
> **このPhaseでの例外：** Phase 1はアプリの初期設定なので、例外として `main` に直接pushしてよい。Phase 2以降は必ずfeatureブランチを使うこと。
>
> 完了したら教えてください。

### 完了確認

- GitHubのリポジトリにコードが反映されていること
- `.env.local` がリポジトリに含まれていないこと（重要）

---

## Step 8 `[USER]`：Vercel にデプロイする

### やること

VercelでGitHubリポジトリを連携し、環境変数を設定してデプロイする。

### Vercel とは

GitHubにpushするだけで自動デプロイしてくれるホスティングサービス。サーバーの設定・管理が不要。

### push-to-deploy の仕組み

1. ローカルで開発・変更
2. GitHubにpush
3. Vercelが自動でビルド・デプロイ
4. 数分で本番URLに反映

手動でのサーバー操作は一切不要。

### ユーザーへの提示内容

> 以下の作業を手動で実施してください。
>
> **① GitHubアプリをVercelにインストールする**
>
> Vercelのダッシュボードから「Add New Project」→「Import Git Repository」に進むと、「Install」ボタンが表示される。クリックするとGitHubの認証画面が開く。
>
> GitHubの認証画面では以下を選択する：
>
> - **Only select repositories** を選択（All repositoriesは不要な権限を与えすぎるので選ばない）
> - `my-folio` リポジトリだけを選択して「Install」をクリック
>
> **② リポジトリをImportする**
>
> Vercelに戻ると `my-folio` がリストに表示されるので「**Import**」をクリックする。
>
> **③ 環境変数を設定する**
>
> 「Environment Variables」セクションを展開し、環境変数を設定する。
>
> > 💡 `.env.local` はドット始まりの隠しファイルのためMacのFinderでは表示されない。`Command + Shift + .` で表示/非表示を切り替えられる。
>
> **方法A：** 「**Import .env**」で `.env.local` をアップロード → そのあと **`DATABASE_URL` だけを手動で上書きする**（Transaction の接続文字列に差し替える）。
>
> **方法B：** 最初から手動で4つを設定する。
>
> | キー                            | 値                                                                          |
> | ------------------------------- | --------------------------------------------------------------------------- |
> | `NEXT_PUBLIC_SUPABASE_URL`      | SupabaseのProject URL                                                       |
> | `NEXT_PUBLIC_SUPABASE_ANON_KEY` | SupabaseのPublishable key                                                   |
> | `SUPABASE_SERVICE_ROLE_KEY`     | SupabaseのSecret key                                                        |
> | `DATABASE_URL`                  | **Transaction モード**の接続文字列（Supabase Connect → Transaction で取得） |
>
> > ⚠️ **重要：** Vercel の `DATABASE_URL` には `.env.local` と同じ Direct connection 文字列をそのまま使ってはいけない。Direct は Vercel から接続できないため、必ず **Transaction** モード（`pooler.supabase.com:6543`）の文字列を使うこと。
>
> **④ Deployする**
>
> 「**Deploy**」ボタンをクリック。1〜2分でデプロイが完了し、公開URLが発行される。
>
> 完了したら教えてください。

### 環境変数の確認方法

デプロイ後、「Continue to Dashboard」→「**Settings → Environment Variables**」から設定済みの環境変数を確認できる。

### プレビューデプロイとは

Vercelはmain以外のブランチをpushしたとき、自動でプレビュー用URLを発行する。本番に影響を与えずに変更を確認できる。Phase 2以降のfeatureブランチ開発で活用する。

### 完了確認

- Vercelのダッシュボードにプロジェクトが表示されていること
- デプロイが成功し、公開URLでアプリが開けること
- 「Settings → Environment Variables」で全環境変数が設定されていること
- **`DATABASE_URL` が Transaction の接続文字列になっていること**（Phase 4 以降で DB 操作する場合に必須）

---

## Step 9 `[AGENT]`：READMEを書く

### やること

プロジェクトのルートにある `README.md` を更新し、最低限の情報を記載する。

### なぜ最初に書くのか

「あとで書こう」は永遠に来ない。3週間後の自分や、プロジェクトを引き継ぐ人が困らないよう、最初に書く習慣をつける。

### 最低限書くべき内容

- プロジェクトの概要（1〜2行）
- ローカルでの起動方法
- 必要な環境変数の一覧（値は書かない、キー名だけ）
- 各Phaseで何を実装したかのメモ

### 確認ポイント

- READMEを見ただけで、初めてクローンした人がローカル起動できるか
- 環境変数のキー名が全部記載されているか

---

## Phase 1 完了チェックリスト

- [ ] `npm run dev` でローカル起動できる
- [ ] shadcn/ui のコンポーネントが `components/ui/` に存在する
- [ ] `prisma/schema.prisma` にデータモデルが定義されている
- [ ] `.env.local` に全環境変数が設定されている
- [ ] `.env.local` が `.gitignore` で除外されている
- [ ] GitHubリポジトリにコードがpushされている
- [ ] VercelのデプロイURLでアプリが開ける
- [ ] Vercelに全環境変数が設定されている（`DATABASE_URL` は Transaction モード）
- [ ] `README.md` に起動方法と環境変数一覧が書かれている

---

## よくある失敗パターン

**環境変数をGitHubにpushしてしまった**
すぐに該当のAPIキーを無効化し、Supabaseやその他のサービスで再発行すること。「削除してもGitHubの履歴には残る」ため、キーの再発行が必須。

**Vercelに環境変数を設定し忘れた**
デプロイは成功するがアプリが正常に動かない。VercelのProject Settingsで環境変数を追加後、再デプロイが必要。

**Vercel の DATABASE_URL に Direct connection を使った**
Direct connection（`db.xxx.supabase.co:5432`）は Vercel のサーバーレス環境から接続できない。`Can't reach database server` や P1001 エラーが出る場合は、Supabase Connect → **Transaction** モードの接続文字列に差し替えること。

**`NEXT_PUBLIC_` をservice_role keyにつけてしまった**
ブラウザに秘密のキーが露出する。すぐに削除し、Supabaseで再発行すること。

---

## 次のPhase

Phase 1が完了したら、Phase 2（認証）に進む。
Phase 2では Supabase Auth を使ってオーナーのログイン機能を実装し、`/admin` 以下のページを保護する。

---

_このガイドはFolio研修カリキュラムの一部です。不明点はメンターに確認してください。_
