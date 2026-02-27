---
title: "TanStack Start の前に知っておきたい TanStack Router の基本"
emoji: "🏝️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tanstack", "tanstackrouter", "tanstackstart"]
published: false
---

# TanStack Start の前に知っておきたい TanStack Router の基本

## はじめに

TanStack Start が RC になってから名前をよく耳にするようになりました。キャッチアップを進める中で、Start を学ぶ前にまず Router をちゃんと理解しておいた方がいいと感じたので整理します。

TanStack Start は TanStack Router の上に構築されたフルスタックフレームワークです。つまり Router を理解していなければ Start も理解できませんし、逆に Router を把握していれば Start は「サーバーレイヤーの追加」として素直に理解できます。

この記事では **TanStack Router 単体の機能**だけを整理します。すべてクライアントサイドで完結する話です。

なお、この記事では **File-based Routing** を前提に解説します。TanStack Router は Code-based Routing にも対応していますが、Start でも File-based Routing が標準なので、こちらに絞ります。

---

## 1. TanStack Router の特徴

TanStack Router は React 向けのルーティングライブラリで、以下のような特徴があります。

- **100% 型安全**: パス、パスパラメータ、Search Params、Context、Loader のすべてに型が付く。存在しないルートへの `<Link>` はコンパイルエラーになる
- **File-based Routing と Code-based Routing の両対応**: File-based の場合は TanStack Router Plugin（Vite / Rspack / Webpack 対応）がルートツリーを自動生成する
- **Search Params を型安全に管理できる**: クエリパラメータにスキーマ定義・バリデーション・デフォルト値をルート定義の一部として組み込める。他のルーターでは文字列を手動パースするのが一般的だが、TanStack Router ではパスパラメータと同じレベルで型が付く
- **フレームワーク非依存**: Router 単体で Vite + React プロジェクトに導入できる。Start は必須ではない

---

## 2. ルートツリーと `__root.tsx`

TanStack Router はルート定義が**ツリー構造**になっています。すべてのルートは一つのルートツリーに属し、その最上位にあるのが `__root.tsx` です。

`__root.tsx` は全ページ共通のレイアウトを定義する場所で、HTML の `<html>` や `<body>` に相当する骨格をここに書きます。

```tsx
// routes/__root.tsx
import { createRootRoute, Outlet } from '@tanstack/react-router'

export const Route = createRootRoute({
  component: () => (
    <div>
      <header>ナビゲーション</header>
      <Outlet />
    </div>
  ),
})
```

`<Outlet />` は、現在の URL にマッチした子ルートのコンポーネントを描画するためのプレースホルダです。

後述する Route Context を使いたい場合は、`createRootRouteWithContext` を使います。

```tsx
import { createRootRouteWithContext } from '@tanstack/react-router'

type AuthState = {
  user: User | null
  isAuthenticated: boolean
}

type RouterContext = {
  auth: AuthState
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootComponent,
})
```

---

## 3. ファイルベースルーティングの規約

`routes/` ディレクトリ内のファイル構成がそのままルートツリーになります。

| ファイルパス | URL パス |
|---|---|
| `routes/index.tsx` | `/` |
| `routes/about.tsx` | `/about` |
| `routes/posts/index.tsx` | `/posts` |
| `routes/posts/$postId.tsx` | `/posts/:postId` |

TanStack Router Plugin（または CLI）がこれらのファイルを検出し、`routeTree.gen.ts` というファイルを自動生成します。このファイルにはすべてのルートの型情報が含まれており、`<Link>` や `useParams` などの型安全を実現する基盤になっています。

フラット構成とディレクトリ構成を混在させることもできます。たとえば `routes/posts.tsx` と `routes/posts/index.tsx` を同時に使うことで、`/posts` のレイアウトとインデックスページを分離できます。

---

## 4. レイアウトの定義方法

TanStack Router のレイアウト定義とルートグループには、目的別にいくつかの仕組みがあります。

### 4-1. Layout Routes（`route.tsx` を使ったレイアウト）

ディレクトリ内に `route.tsx` を置くと、そのパスに対応する**レイアウトルート**になります。

```
routes/
├── posts/
│   ├── route.tsx      ← /posts のレイアウト（<Outlet /> を持つ）
│   ├── index.tsx      ← /posts のコンテンツ
│   └── $postId.tsx    ← /posts/:postId のコンテンツ
```

```tsx
// routes/posts/route.tsx
export const Route = createFileRoute('/posts')({
  component: () => (
    <div>
      <h1>記事</h1>
      <Outlet />  {/* index.tsx や $postId.tsx がここに描画される */}
    </div>
  ),
})
```

URL パスに対応し、`<Outlet />` を通じて子ルートを描画します。

### 4-2. Pathless Layout Routes（`_` プレフィックス）

ファイル名の先頭に `_` を付けると、**URL パスに影響しないレイアウトルート**になります。URL には現れないが、コンポーネントツリーには影響します。

```
routes/
├── _layout.tsx           ← レイアウト定義（URL には現れない）
├── _layout/
│   ├── dashboard.tsx     ← /dashboard
│   └── settings.tsx      ← /settings
```

```tsx
// routes/_layout.tsx
export const Route = createFileRoute('/_layout')({
  component: () => (
    <div className="app-layout">
      <Sidebar />
      <main>
        <Outlet />
      </main>
    </div>
  ),
})
```

この例では `/dashboard` と `/settings` にアクセスすると `_layout.tsx` のレイアウトが適用されますが、URL には `_layout` は一切現れません。

サイドバー付きレイアウト、認証済みエリアのラッパー、管理画面のレイアウトなど、特定のルート群だけを共通のレイアウトでラップしたいケースに使います。

### 4-3. Pathless Route Group Directories（`()` ディレクトリ）

ディレクトリ名を `()` で囲むと、**純粋なファイル整理用のグルーピング**になります。

```
routes/
├── index.tsx
├── (app)/
│   ├── dashboard.tsx    ← /dashboard
│   ├── settings.tsx     ← /settings
│   └── users.tsx        ← /users
├── (auth)/
│   ├── login.tsx        ← /login
│   └── register.tsx     ← /register
```

`(app)` や `(auth)` はルートツリーにもコンポーネントツリーにも**一切影響しません**。ファイルの数が増えたときに整理するためだけの仕組みです。レイアウトの適用はできません。

---

## 5. レイアウトの仕組み — Next.js との違い

セクション 4 で紹介した仕組みは、Next.js 経験者が最も混乱しやすいポイントです。**最大の違いは `()` の役割です。**

Next.js の Route Groups `()` は、ファイル整理に加えて `layout.tsx` を配置すればレイアウトの適用もできます。つまり「ファイル整理 + レイアウト適用」の両方を兼ねています。

TanStack Router の `()` は**ファイル整理のみ**です。レイアウトを適用したい場合は `_` プレフィックス（Pathless Layout Routes）を使う必要があります。

| | TanStack Router | Next.js |
|---|---|---|
| `()` の役割 | ファイル整理**のみ** | ファイル整理 **+** `layout.tsx` でレイアウト適用 |
| URL に影響しないレイアウト | `_` プレフィックス | `()` 内に `layout.tsx` を配置 |
| ディレクトリごとの `layout.tsx` | なし（`route.tsx` で代替） | あり（各ディレクトリに配置可能） |

Next.js では `()` がファイル整理とレイアウト適用の両方を兼ねていますが、TanStack Router では `()` は整理専用、`_` がレイアウト専用と、**役割が明確に分離**されています。

---

## 6. ナビゲーション

### `<Link>` コンポーネント

TanStack Router の `<Link>` は**型安全**です。`to` プロパティにはルートツリーに存在するパスしか指定できず、存在しないパスはコンパイルエラーになります。

```tsx
import { Link } from '@tanstack/react-router'

// 静的なパス
<Link to="/about">About</Link>

// 動的パラメータ付き（params も型で強制される）
<Link to="/posts/$postId" params={{ postId: '1' }}>
  記事を見る
</Link>
```

### `useNavigate`

プログラム的な遷移には `useNavigate` を使います。

```tsx
import { useNavigate } from '@tanstack/react-router'

const navigate = useNavigate()

navigate({ to: '/posts/$postId', params: { postId: '1' } })
```

---

## 7. パスパラメータ

動的な URL セグメントは、ファイル名に `$` プレフィックスを付けて定義します。

```
routes/posts/$postId.tsx  →  /posts/:postId
```

ルートコンポーネント内では `Route.useParams()` で型付きのパラメータを取得できます。

```tsx
// routes/posts/$postId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  component: PostPage,
})

function PostPage() {
  const { postId } = Route.useParams()
  // postId の型は string として推論される
  return <div>Post: {postId}</div>
}
```

型は `routeTree.gen.ts` から自動推論されるため、手動で型定義を書く必要がありません。

---

## 8. Search Params

Search Params（クエリパラメータ）の扱いは **TanStack Router の最大の差別化ポイント**の一つです。`?page=2&sort=date` のようなクエリパラメータを、**型付きの状態**として管理できます。

### 定義

ルート定義の `validateSearch` でスキーマを定義します。

```tsx
// routes/posts/index.tsx
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

const postsSearchSchema = z.object({
  page: z.number().default(1),
  sort: z.enum(['date', 'title']).default('date'),
  filter: z.string().optional(),
})

export const Route = createFileRoute('/posts/')({
  validateSearch: postsSearchSchema,
  component: PostsPage,
})
```

### 取得

`Route.useSearch()` で型付きの値を取得できます。

```tsx
function PostsPage() {
  const { page, sort, filter } = Route.useSearch()
  // page: number, sort: 'date' | 'title', filter: string | undefined
  // デフォルト値も適用済み
}
```

### 設定

`<Link>` の `search` プロパティで型安全に設定できます。

```tsx
<Link to="/posts" search={{ page: 2, sort: 'title' }}>
  2ページ目（タイトル順）
</Link>
```

一般的なルーターでは `useSearchParams()` で文字列を手動パースする必要がありますが、TanStack Router では Search Params が**ルート定義の一部**として管理されるため、型・バリデーション・デフォルト値がすべてルート定義に含まれます。

---

## 9. Route Context

Route Context は、**ルートツリーを通じてデータを伝搬する仕組み**です。認証情報やサービスクライアントなど、複数のルートで共有したいデータをルートツリーに沿って型安全に渡せます。

### 定義

まず `__root.tsx` で Context の型を定義します。

```tsx
// routes/__root.tsx
import { createRootRouteWithContext } from '@tanstack/react-router'

type RouterContext = {
  auth: {
    user: User | null
    isAuthenticated: boolean
  }
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootComponent,
})
```

ルーター定義時にはプレースホルダを渡しておきます。

```tsx
// router.tsx
const router = createRouter({
  routeTree,
  context: {
    auth: undefined!, // 実際の値は RouterProvider で渡す
  },
})
```

`undefined!` は TypeScript に「ここでは型だけ見てくれ」と伝えるイディオムです。実際の値は `<RouterProvider>` の `context` プロパティで注入します。こうすることで React のフック（認証ライブラリの `useAuth()` など）から取得した値を渡せます。

```tsx
// App.tsx
function App() {
  const auth = useAuth() // Clerk, Auth0, Firebase Auth など
  return <RouterProvider router={router} context={{ auth }} />
}
```

### 利用

`beforeLoad` はルート遷移前に実行されるフックです。context にアクセスして認証チェックを行ったり、後続のルートに追加データを注入したりできます。

リダイレクトが必要な場合は `redirect` を `throw` します。`return` ではなく `throw` することで、TanStack Router がそれを遷移指示として受け取ります。

```tsx
import { createFileRoute, redirect } from '@tanstack/react-router'

// routes/dashboard.tsx
export const Route = createFileRoute('/dashboard')({
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' })
    }
  },
  component: DashboardPage,
})
```

コンポーネント内では `Route.useRouteContext()` で取得します。

```tsx
function DashboardPage() {
  const { auth } = Route.useRouteContext()
  return <div>Welcome, {auth.user?.name}</div>
}
```

---

## 10. Loader

Loader は**ルート遷移前にデータを取得する仕組み**です。ユーザーがページに到達する前にデータを準備しておけるため、コンポーネント内で `useEffect` を使ったデータ取得よりもスムーズな体験を提供できます。

### 基本的な使い方

```tsx
// routes/posts/$postId.tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId)
    return { post }
  },
  component: PostPage,
})

function PostPage() {
  const { post } = Route.useLoaderData()
  return <h1>{post.title}</h1>
}
```

`loader` 関数内では `params`、`search`（Search Params）、`context`（Route Context）にアクセスできます。ページネーションやフィルタリングが必要な場合は `search` を使ってデータ取得に渡せます。

```tsx
loader: async ({ params, context }) => {
  // params, context はすべて型付き
  const post = await fetchPost(params.postId)
  return { post, currentUser: context.auth.user }
}
```

### 重要：Router の Loader はクライアントサイドで動く

**TanStack Router 単体の Loader は、純粋にクライアントサイド（ブラウザ上）で実行されます。**

Server Functions と組み合わせて Loader をサーバーサイドで実行する仕組みは **TanStack Start の機能**であり、Router 単体の範囲ではありません。

---

## 11. Pending / Error / NotFound

TanStack Router では、各ルートに対して **Pending UI**、**Error UI**、**Not Found UI** をルート定義のプロパティとして設定できます。

### pendingComponent

Loader の実行中に表示するフォールバック UI です。

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => fetchPost(params.postId),
  pendingComponent: () => <div>読み込み中...</div>,
  component: PostPage,
})
```

### errorComponent

ルートの Loader やコンポーネントでエラーが発生した場合に表示されます。ルート単位のエラーバウンダリとして機能します。

```tsx
export const Route = createFileRoute('/posts/$postId')({
  errorComponent: ({ error }) => (
    <div>エラーが発生しました: {error.message}</div>
  ),
  // ...
})
```

### notFoundComponent

存在しないルートにアクセスした場合の表示です。

```tsx
// routes/__root.tsx でデフォルトを定義
export const Route = createRootRoute({
  notFoundComponent: () => <div>ページが見つかりません</div>,
  component: RootComponent,
})
```

これらは各ルートで個別に設定することも、`__root.tsx` でデフォルトを定義しておくこともできます。

---

## まとめ：Router と Start の境界

この記事で解説した内容は、すべて**クライアントサイドで完結する TanStack Router の機能**です。

| Router の機能（この記事の範囲） | Start が追加する機能 |
|---|---|
| File-based Routing | SSR / SSG |
| 型安全なナビゲーション | Server Functions (`createServerFn`) |
| パスパラメータ / Search Params | サーバーサイド Loader |
| Route Context | API Routes |
| クライアントサイド Loader | サーバーサイドの Context 注入 |
| レイアウト / Pathless Routes | ストリーミング SSR |
| Pending / Error / NotFound | — |

Router を把握していれば、Start の学習は「ここにサーバーレイヤーが加わるのか」という視点で進められます。Router の知識はそのまま Start でも活きるので、まずは Router の機能をしっかり理解しておくことをおすすめします。

---

## 参考

- [TanStack Router 公式ドキュメント](https://tanstack.com/router/latest/docs/framework/react/overview)
- [TanStack Start 公式ドキュメント](https://tanstack.com/start/latest/docs/framework/react/overview)
