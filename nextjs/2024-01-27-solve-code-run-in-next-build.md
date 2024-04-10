---
category: nextjs
author: forhot2000@qq.com
date: 2024/01/27
---

# 解决 next build 阶段执行代码的问题

今天遇到一个头疼的问题，花了几天时间把功能写好了，在 dev 模式下运行正常，然而 build 的时候却意外发现 nextjs 竟然访问数据库了。

简化下代码，仅拿出问题相关的代码。

```js
// ./src/app/page.js
export default async function Page() {
    const data = await getData(); // next build 调用了 getData
    return ...
}
```

检查了一遍项目中的代码，确认没有在代码文件中硬编码写入访问数据库的代码，那是什么原因造成的呢？

在 pages router 模式下并没有遇到这个问题，按照 pages router 模式推荐的使用 getServerSideProps 方法提供数据，在 build 阶段不会调用到 getData 方法。

```js
// ./src/pages/index.js
export default function Page({data}) {
    return ...
}

export const getServerSideProps = async () => {
    const data = await getData(); // getData 仅在访问页面时被调用
    return { props: { data } };
};
```

根据这个情况，我找到了 app router 的[迁移指导][1]，里面提到了设置 fetch 的 cache 参数为 'no-store' 即可达到与 getServerSideProps 一样的效果。

```js
const res = await fetch(`https://...`, { cache: "no-store" });
```

但是，我是直接从数据库取数据的，所以这个方法不适合我，再找下缓存设置，发现可以使用 [unstable_noStore][3] 来禁用缓存。

```js
// ./src/app/page.js
import { unstable_noStore as noStore } from 'next/cache';

export default async function Page() {
    noStore();
    const data = await getData(); // getData 仅在访问页面时被调用
    return ...
}
```

到这里，问题基本上已经定位准确了，但是看到那个 unstable 就觉得还不稳，于是继续往下查，文档中提到当你使用第三方库获取数据时，在页面 export 一个 [dynamic][4] 变量将页面设置为 dynamic，这样也等同于使用 getServerSideProps 方法获取数据。

```js
// ./src/app/page.js

// 将当前页面设置为动态页面
export const dynamic = 'force-dynamic';

export default async function Page() {
    const data = await getData(); // getData 仅在访问页面时被调用
    return ...
}
```

为了避免每个页面都去加 dynamic 设置，可以在 layout 上加这个设置，这样可以作用到每个页面上。

```js
// ./src/app/layout.js

// 将所有使用当前 layout 的页面设置为动态页面
export const dynamic = 'force-dynamic';

export default async function Layout() {
    return ...
}
```

好了，到这里问题就算是完美解决了。可怜忙了一整天，最后就产出了一行代码~

参考链接：

- [Server-side Rendering (getServerSideProps)][1]
- [unstable_noStore API][3]
- [Fetching data on the Server with third-party libraries][2]
- [route-segment-config#dynamic][4]

[1]: https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration#server-side-rendering-getserversideprops
[2]: https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating#fetching-data-on-the-server-with-third-party-libraries
[3]: https://nextjs.org/docs/app/api-reference/functions/unstable_noStore
[4]: https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#dynamic
