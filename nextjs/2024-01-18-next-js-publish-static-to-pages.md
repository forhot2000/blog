---
category: nextjs
author: forhot2000@qq.com
date: 2024/01/18
---

# Next.js 发布静态网站到 Pages

## 配置构建静态网站

配置 next build 生成静态网站，设置 output 为 export，修改 `next.config.js`

```js
// next.config.js

const nextConfig = {
  ...
  output: 'export',
  distDir: 'dist',
};

module.exports = nextConfig;
```

## 动态路由

对于动态路由的页面 ，我们还需要告诉 Next.js 需要生成那些页面。

App Router 模式下，我们需要添加 `generateStaticParams` 方法。

```js
// /src/app/posts/[slug]/page.js

export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({
    slug: post.slug,
  }));
}

export default function Page({ params }) {
  const { slug } = params;
  // ...
}
```

Pages Router 模式下，则是应该添加 `getStaticPaths` 方法。

```js
// /pages/posts/[slug].tsx

export const getStaticPaths = async () => {
  const posts = await getPosts();
  return {
    paths: [
      {
        params: posts.map((post) => ({
          slug: post.slug,
        })),
      },
    ],
    fallback: true,
  };
};

export async function getStaticProps({ params }) {
  const post = await getPostBySlug(params.slug);
  return { props: { post } };
}

export default function Page({ post }) {
  // ...
}
```

## 配置 basePath

Pages 默认生成的链接是一个带有仓库名称的二级目录链接，为了正常访问页面，我们需要设定 Next.js 的 basePath 设置，修改 `next.config.js`

```js
// next.config.js

const nextConfig = {
  ...
  basePath: '/slqm', // 与你的仓库名称匹配
};

module.exports = nextConfig;
```

如果你使用了与用户名相同的仓库名， Pages 可以生成不带二级目录的链接，那么你就可以不需要配置 basePath 了。

## 构建

配置好了之后，可以运行 `npm run build` 生成静态网站到 dist 目录

## 发布

接下来我们就可以把 dist 目录发布到 Pages 或其它托管平台了。

先安装 gh-pages 包

```shell
npm install --save-dev gh-pages
```

然后，在 package.json 中添加 pages 脚本

```json
{
  "scripts": {
    ...
    "pages": "gh-pages -d dist --nojekyll"
  },
}
```

配置好之后，我们就只需要运行 `npm run pages` 就可以将 dist 目录 push 到 gh-pages 分支了。

为 gh-pages 分支启用 Pages，当新内容 push 到 gh-pages 分支后，GitHub Pages 会自动发布新的内容，而 Gitee Pages 还需要手动到设置页面点下更新发布，或者升级到付费版才能自动更新。
