---
category: nextjs
author: forhot2000@qq.com
date: 2024/01/17
---

# Next.js App Router 中的页面路由

Next.js 从 v13 版本开始加入了 App Router 模式，和原来的 Pages Router 不一样，新的 App Router 将所有路由规则都放在 `/src/app` 目录下，每一个页面对应一个 `page.jsx` 文件，这个文件需要 export 一个 default 的 React 组件，同时还支持 `page.js`, `page.ts`, `page.tsx` 这几个文件名，page 文件所在的目录则对应了它的路由规则。

# 路由规则

路由规则可以参考下表：

| type      | router                                | matches        | params                          |
| --------- | ------------------------------------- | -------------- | ------------------------------- |
| static    | /page.jsx                             | /              | { }                             |
|           | /foo/page.jsx                         | /foo           | { }                             |
|           | /foo/bar/page.jsx                     | /foo/bar       | { }                             |
| dynamic   | /pages/`[slug]`/page.jsx              | /pages/foo     | {`slug`:'foo'}                  |
|           | /pages/`[slug]`/`[sub_slug]`/page.jsx | /pages/foo/bar | {`slug`:'foo',`sub_slug`:'bar'} |
| catch-all | /pages/`[...path]`/page.jsx           | /pages/foo     | {`path`:['foo']}                |
|           |                                       | /pages/foo/bar | {`path`:['foo','bar']}          |
| optional  | /pages/`[[...path]]`/page.jsx         | /pages         | { }                             |
|           |                                       | /pages/foo     | {`path`:['foo']}                |
|           |                                       | /pages/foo/bar | {`path`:['foo','bar']}          |

# 如何获取 params

```jsx
// page.jsx
export default function Page({ params }) {
  const { slug } = params;
  ...
}
```

```tsx
// page.tsx
export default function Page({ params }: { params: { slug: string } }) {
  const { slug } = params;
  ...
}
```
