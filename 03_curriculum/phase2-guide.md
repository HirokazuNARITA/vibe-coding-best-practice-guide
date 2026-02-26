# Phase 2 手順書：認証のセットアップ

> **ゴール：** `/admin` にログインしないとアクセスできない状態にする  
> **所要時間の目安：** 2〜3時間  
> **前提：** Phase 1 が完了していること

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

## Step 1：オーナーアカウントを作成する

### やること

Supabaseダッシュボードから管理者（オーナー）のアカウントを手動で作成する。

### なぜコードで作らないのか

このアプリのオーナーは自分1人だけ。サインアップ機能を作ると誰でもアカウントを作れてしまう。Supabaseのダッシュボードから直接作成することで、余計な機能を実装せずに済む。

### 手順

1. Supabaseダッシュボード →「Authentication」→「Users」を開く
2. 「Add user」→「Create new user」をクリック
3. メールアドレスとパスワードを入力して作成
4. 作成したメールアドレスとパスワードをメモしておく（`.env.local` には入れない）

### 確認ポイント

- Usersにアカウントが表示されていること

---

## Step 2：Supabase SSRパッケージをインストールする

### やること

Next.js App RouterでSupabase Authを正しく使うためのパッケージをインストールする。

### なぜ `@supabase/ssr` が必要なのか

Next.js App RouterはServer ComponentsとClient Componentsが混在する。それぞれでセッションを正しく取得するには `@supabase/ssr` が必要。Phase 1でインストール済みの場合はスキップ。

```bash
npm install @supabase/ssr
```

---

## Step 3：Supabaseクライアントを整備する

### やること

Server Components用・Client Components用・middleware用の3種類のクライアントを用意する。

### なぜ3種類必要なのか

| 用途                | 理由                                     |
| ------------------- | ---------------------------------------- |
| Server Components用 | サーバー側でセッションを読み取るため     |
| Client Components用 | ブラウザ側でセッションを管理するため     |
| middleware用        | リクエスト単位でセッションを検証するため |

### `lib/supabase.ts`（Client Components用）

Phase 1で作成済みのため変更不要。

```ts
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
}
```

### `lib/supabase-server.ts`（Server Components用）

新しく作成する。

```ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createServerSupabaseClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => {
            cookieStore.set(name, value, options);
          });
        },
      },
    },
  );
}
```

### `lib/supabase-middleware.ts`（middleware用）

新しく作成する。

```ts
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value),
          );
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options),
          );
        },
      },
    },
  );

  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user && !request.nextUrl.pathname.startsWith('/admin/login')) {
    const url = request.nextUrl.clone();
    url.pathname = '/admin/login';
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}
```

### 確認ポイント

- `lib/` に3つのファイルが存在すること

---

## Step 4：middleware.ts を実装する

### やること

`/admin` 以下へのアクセスを認証で保護する。未認証ユーザーは `/admin/login` にリダイレクトされる。

### `middleware.ts`（プロジェクトルートに配置）

```ts
import { type NextRequest } from 'next/server';
import { updateSession } from '@/lib/supabase-middleware';

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: ['/admin/:path*'],
};
```

### ポイント

- `matcher` に `/admin/:path*` を指定することで `/admin` 以下のみに適用される
- `/admin/login` 自体は除外しないと無限リダイレクトになるため、`updateSession` 内で除外している

### 確認ポイント

- ブラウザで `/admin` にアクセスすると `/admin/login` にリダイレクトされること

---

## Step 5：ログイン画面を実装する

### やること

`/admin/login` にメール・パスワードのログインフォームを作成する。

### `app/admin/login/page.tsx`

```tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);
  const router = useRouter();
  const supabase = createClient();

  const handleLogin = async () => {
    setLoading(true);
    setError(null);

    const { error } = await supabase.auth.signInWithPassword({
      email,
      password,
    });

    if (error) {
      setError('メールアドレスまたはパスワードが正しくありません');
      setLoading(false);
      return;
    }

    router.push('/admin');
    router.refresh();
  };

  return (
    <div className='min-h-screen flex items-center justify-center'>
      <div className='w-full max-w-sm space-y-6 p-8'>
        <h1 className='text-2xl font-bold text-center'>管理者ログイン</h1>
        <div className='space-y-4'>
          <div className='space-y-2'>
            <Label htmlFor='email'>メールアドレス</Label>
            <Input
              id='email'
              type='email'
              value={email}
              onChange={(e) => setEmail(e.target.value)}
            />
          </div>
          <div className='space-y-2'>
            <Label htmlFor='password'>パスワード</Label>
            <Input
              id='password'
              type='password'
              value={password}
              onChange={(e) => setPassword(e.target.value)}
            />
          </div>
          {error && <p className='text-sm text-red-500'>{error}</p>}
          <Button
            className='w-full'
            onClick={handleLogin}
            disabled={loading}
          >
            {loading ? 'ログイン中...' : 'ログイン'}
          </Button>
        </div>
      </div>
    </div>
  );
}
```

### 確認ポイント

- `/admin/login` にアクセスするとログインフォームが表示されること
- 正しい認証情報でログインすると `/admin` にリダイレクトされること
- 間違った認証情報ではエラーメッセージが表示されること

---

## Step 6：ログアウト機能を実装する

### やること

管理画面にログアウトボタンを追加する。

### `app/admin/page.tsx` に追加

```tsx
import { redirect } from 'next/navigation';
import { createServerSupabaseClient } from '@/lib/supabase-server';
import { LogoutButton } from '@/components/logout-button';

export default async function AdminPage() {
  const supabase = await createServerSupabaseClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) redirect('/admin/login');

  return (
    <div className='p-8'>
      <div className='flex justify-between items-center mb-8'>
        <h1 className='text-2xl font-bold'>管理ダッシュボード</h1>
        <LogoutButton />
      </div>
      <p className='text-muted-foreground'>
        作品管理機能はPhase 3以降で実装します。
      </p>
    </div>
  );
}
```

### `components/logout-button.tsx`

```tsx
'use client';

import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase';
import { Button } from '@/components/ui/button';

export function LogoutButton() {
  const router = useRouter();
  const supabase = createClient();

  const handleLogout = async () => {
    await supabase.auth.signOut();
    router.push('/admin/login');
    router.refresh();
  };

  return (
    <Button
      variant='outline'
      onClick={handleLogout}
    >
      ログアウト
    </Button>
  );
}
```

### 確認ポイント

- ログアウトボタンをクリックすると `/admin/login` にリダイレクトされること
- ログアウト後に `/admin` にアクセスすると `/admin/login` にリダイレクトされること

---

## Phase 2 完了チェックリスト

- [ ] Supabaseダッシュボードにオーナーアカウントが存在する
- [ ] `/admin` にアクセスすると `/admin/login` にリダイレクトされる
- [ ] 正しい認証情報でログインできる
- [ ] ログイン後に `/admin` ダッシュボードが表示される
- [ ] ログアウトボタンで `/admin/login` にリダイレクトされる
- [ ] featureブランチで開発してPR→mainにマージした

---

## よくある失敗パターン

**ログイン後に画面が更新されない**
`router.refresh()` を忘れている。Next.jsのキャッシュをクリアするために必要。

**`/admin/login` にアクセスしても無限リダイレクトになる**
`updateSession` 内の `/admin/login` 除外条件を確認すること。

**Server Componentでセッションが取得できない**
`lib/supabase.ts`（Client用）を誤ってServer Componentで使っている。Server Componentでは `lib/supabase-server.ts` を使うこと。

---

## 次のPhase

Phase 2が完了したら、Phase 3（UI構築）に進む。  
Phase 3では shadcn/ui を使ってトップページ・作品詳細ページ・管理ダッシュボードのUIを実装する。

---

_このガイドはFolio研修カリキュラムの一部です。不明点はメンターに確認してください。_
