# Phase 4 手順書：フォーム・DB

> **ゴール：** 作品のCRUD（作成・読取・更新・削除）が実際のDBで動く状態にする  
> **所要時間の目安：** 3〜4時間  
> **前提：** Phase 3 が完了していること

---

## エージェント向け指示

このドキュメントの各Stepには実施者ラベルが付いています。

- **`[USER]`** — ユーザーが手動で実施する作業（ダッシュボード操作・アカウント作成など）。エージェントはこのStepの「ユーザーへの提示内容」をユーザーにそのまま提示し、完了確認を待ってから次のStepに進むこと。
- **`[AGENT]`** — エージェントが実施する作業（コード実装・コマンド実行など）。エージェントが自律的に実行すること。

---

## このPhaseで学ぶこと

- **Prisma** を使ったスキーマ定義・マイグレーション・DB操作
- **Zod** を使ったバリデーションスキーマの定義
- **React Hook Form + Zod** を組み合わせたフォーム実装
- **Server Actions** を使ったサーバーサイドのDB操作
- **REST APIをゼロから作らない** という原則の体験

---

## このPhaseで使うツール

| ツール          | 役割                                          | 料金        |
| --------------- | --------------------------------------------- | ----------- |
| Prisma          | ORM（スキーマ定義・マイグレーション・DB操作） | 無料（OSS） |
| Zod             | バリデーションスキーマ定義                    | 無料（OSS） |
| React Hook Form | フォーム状態管理                              | 無料（OSS） |

---

## このPhaseで実装すること

- [ ] Prismaスキーマに `Work` モデルを定義してマイグレーション実行
- [ ] Server Actions（`actions/works.ts`）の実装
- [ ] 作品投稿フォーム（`/admin/works/new`）の実装
- [ ] 作品編集フォーム（`/admin/works/[id]/edit`）の実装
- [ ] 削除機能の実装
- [ ] 公開・非公開の切り替え
- [ ] Phase 3のダミーデータをDBからの実データに差し替え

---

## Server Actions とは

このPhaseで最も重要な概念。REST APIを作らずにサーバーサイドのDB操作を行う仕組み。

| 従来のやり方                         | Server Actionsを使う場合            |
| ------------------------------------ | ----------------------------------- |
| `/api/works` エンドポイントを作成    | `actions/works.ts` に関数を書くだけ |
| fetch でPOST/PUT/DELETEリクエスト    | 関数を直接呼び出す                  |
| リクエスト・レスポンスの型定義が必要 | TypeScriptの型がそのまま使える      |

**原則：** DBへの書き込みはすべてServer Actionsで実装する。REST APIはゼロから作らない。

---

## Step 1 `[AGENT]`：必要なパッケージをインストールする

### やること

このPhaseで使うパッケージをインストールする。

### インストールするパッケージ

- `zod` — バリデーション
- `react-hook-form` — フォーム管理
- `@hookform/resolvers` — React Hook FormとZodを繋ぐアダプター

### 確認ポイント

- `package.json` に上記3つが含まれていること

---

## Step 2 `[AGENT]`：Prismaスキーマを定義してマイグレーションを実行する

### やること

`prisma/schema.prisma` に `Work` モデルを定義し、Supabaseのデータベースにテーブルを作成する。

### Work モデルのフィールド（仕様書より）

| フィールド  | 型            | 説明                          |
| ----------- | ------------- | ----------------------------- |
| id          | String (cuid) | 主キー・自動生成              |
| title       | String        | タイトル（必須）              |
| description | String        | 説明（必須）                  |
| category    | String        | カテゴリ（必須）              |
| imageUrl    | String        | 画像URL（Phase 5で使用）      |
| price       | Int           | 価格（0以上）                 |
| isPublished | Boolean       | 公開設定（デフォルト: false） |
| createdAt   | DateTime      | 作成日時・自動生成            |
| updatedAt   | DateTime      | 更新日時・自動更新            |

### マイグレーションの実行

スキーマを定義したら以下のコマンドでSupabaseのDBにテーブルを作成する。

```bash
npx prisma migrate dev --name add-work-model
```

### エージェントへの指示例

> `prisma/schema.prisma` に Work モデルを定義してください。フィールドは folio-spec.md のデータモデルセクションを参照してください。定義後に `npx prisma migrate dev --name add-work-model` を実行してください。

### 確認ポイント

- Supabaseダッシュボードの「Table Editor」に `Work` テーブルが作成されていること
- `prisma/migrations/` フォルダが作成されていること

---

## Step 3 `[AGENT]`：Zodバリデーションスキーマを定義する

### やること

フォームのバリデーションルールをZodで定義する。Server Actionsでも同じスキーマを使うことで、フロントエンドとサーバーサイドで一貫したバリデーションが実現できる。

### バリデーションルール（仕様書より）

| フィールド  | ルール                                        |
| ----------- | --------------------------------------------- |
| title       | 必須・最大100文字                             |
| description | 必須・最大1000文字                            |
| category    | 必須                                          |
| imageUrl    | 必須（Phase 5までは空文字列を許容してもよい） |
| price       | 0以上の整数                                   |
| isPublished | Boolean                                       |

### 定義する場所

`actions/works.ts` 内、またはスキーマ専用ファイル `lib/validations/work.ts` として定義する。

### エージェントへの指示例

> 作品フォームのZodバリデーションスキーマを定義してください。フィールドとルールは folio-spec.md を参照してください。

### 確認ポイント

- Zodスキーマが定義されていること

---

## Step 4 `[AGENT]`：Server Actions を実装する

### やること

`actions/works.ts` に作品のCRUD操作をServer Actionsとして実装する。

### 実装する関数

| 関数名          | 処理                                                 |
| --------------- | ---------------------------------------------------- |
| `getWorks`      | 作品一覧を取得（公開ページ用：isPublished=trueのみ） |
| `getAllWorks`   | 全作品を取得（管理画面用：非公開含む）               |
| `getWorkById`   | IDで1件取得                                          |
| `createWork`    | 作品を作成                                           |
| `updateWork`    | 作品を更新                                           |
| `deleteWork`    | 作品を削除                                           |
| `togglePublish` | 公開・非公開を切り替え                               |

### ポイント

- 各関数の先頭に `"use server"` を書くこと
- 書き込み系の関数（create/update/delete）はZodでバリデーションを行う
- 書き込み後は `revalidatePath` で該当ページのキャッシュを更新する

### エージェントへの指示例

> `actions/works.ts` に作品のServer Actionsを実装してください。getWorks・getAllWorks・getWorkById・createWork・updateWork・deleteWork・togglePublish の7つの関数を実装してください。書き込み系はZodバリデーションを行い、revalidatePathでキャッシュを更新してください。

### 確認ポイント

- `actions/works.ts` に7つの関数が実装されていること

---

## Step 5 `[AGENT]`：作品投稿フォームを実装する

### やること

`app/admin/works/new/page.tsx` と `components/work-form.tsx` を実装する。

### 要件

- `components/work-form.tsx` は投稿・編集の両方で使い回せるコンポーネントにする
- React Hook Form + Zod でフォームを実装する（Client Component）
- フィールド：タイトル・説明・カテゴリ・価格・公開設定
- 画像フィールドはPhase 5で追加するため、このPhaseでは省略してよい
- 送信時に `createWork` Server Actionを呼び出す
- 送信成功後は `/admin` にリダイレクトする
- 送信中はボタンをローディング状態にする
- shadcn/ui の `Form` コンポーネントを使う

### エージェントへの指示例

> `components/work-form.tsx` を実装してください。React Hook Form + Zodを使ったClient Componentで、投稿・編集の両方で使い回せる設計にしてください。フィールドはタイトル・説明・カテゴリ・価格・公開設定です（画像はPhase 5で追加）。送信時にcreateWork/updateWorkを呼び出します。

### 確認ポイント

- `/admin/works/new` にアクセスするとフォームが表示されること
- フォームを送信すると作品がDBに保存されること
- バリデーションエラーがフォーム上に表示されること
- 送信成功後に `/admin` にリダイレクトされること

---

## Step 6 `[AGENT]`：作品編集フォームを実装する

### やること

`app/admin/works/[id]/edit/page.tsx` を実装する。

### 要件

- Server Componentとして実装し、DBから既存データを取得する
- 取得したデータを `WorkForm` の初期値として渡す
- 送信時に `updateWork` Server Actionを呼び出す

### エージェントへの指示例

> `app/admin/works/[id]/edit/page.tsx` を実装してください。Server ComponentでDBから既存データを取得し、WorkFormに初期値として渡してください。送信時はupdateWorkを呼び出します。

### 確認ポイント

- 編集ページに既存データが初期値として表示されること
- 編集後に保存するとDBが更新されること

---

## Step 7 `[AGENT]`：削除機能を実装する

### やること

管理ダッシュボードの削除ボタンに `deleteWork` Server Actionを紐づける。

### 要件

- 削除前に確認ダイアログ（shadcn/ui の `Dialog`）を表示する
- 確認後に `deleteWork` を呼び出す
- 削除後は一覧が自動で更新される（`revalidatePath` で対応済みのため自動）

### エージェントへの指示例

> 管理ダッシュボードの削除ボタンにshadcn/uiのDialogを使った確認ダイアログを追加し、確認後にdeleteWork Server Actionを呼び出してください。

### 確認ポイント

- 削除ボタンをクリックすると確認ダイアログが表示されること
- 確認後に作品が削除されること

---

## Step 8 `[AGENT]`：ダミーデータを実データに差し替える

### やること

Phase 3で使っていたダミーデータを、Server Actionsからの実データに差し替える。

### 対象ファイル

- `app/(public)/page.tsx` → `getWorks()` を使う
- `app/(public)/works/[id]/page.tsx` → `getWorkById()` を使う
- `app/admin/page.tsx` → `getAllWorks()` を使う

### エージェントへの指示例

> トップページ・作品詳細ページ・管理ダッシュボードのダミーデータを、Server Actionsからの実データに差し替えてください。

### 確認ポイント

- 管理画面から追加した作品がトップページに表示されること
- 非公開の作品がトップページに表示されないこと

---

## Phase 4 完了チェックリスト

- [ ] Supabaseに `Work` テーブルが作成されていること
- [ ] 作品の新規作成ができること
- [ ] 作品の編集ができること
- [ ] 作品の削除（確認ダイアログ付き）ができること
- [ ] 公開・非公開の切り替えができること
- [ ] トップページに公開作品だけが表示されること
- [ ] フォームのバリデーションが機能していること
- [ ] featureブランチで開発してPR→mainにマージした

---

## よくある失敗パターン

**マイグレーション後にSupabaseのRLSでDB操作がブロックされる**
Phase 1でautomatic RLSをONにしているため、マイグレーションで作成されたテーブルにRLSが適用されている。Server ActionsからPrismaでアクセスする場合は `DATABASE_URL`（直接接続）を使うため通常は問題ないが、エラーが出た場合はSupabaseダッシュボードのRLS設定を確認すること。

**revalidatePath を忘れて画面が更新されない**
create/update/delete後に `revalidatePath` を呼ばないと、Next.jsのキャッシュが残って古いデータが表示される。エージェントへの指示に「revalidatePathを必ず呼ぶ」と明記すること。

**Client Componentでprismaを直接使おうとする**
PrismaはServer Componentまたはサーバーサイドのコード（Server Actions）からのみ使用できる。クライアントサイドでDBを直接操作しようとするとエラーになる。

---

## 次のPhase

Phase 4が完了したら、Phase 5（ファイルアップロード）に進む。  
Phase 5ではUploadThingをセットアップして、作品フォームに画像アップロード機能を追加する。

---

_このガイドはFolio研修カリキュラムの一部です。不明点はメンターに確認してください。_
