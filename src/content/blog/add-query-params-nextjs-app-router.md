---
title: 'Add query Params in NextJS App router'
description: 'With the release of the App router in NextJS 13, we lost the ability to use next/router shallow property. Subsequently, we also lost the ability to use the shallow property to prevent a page from reloading when adding query parameters to the page URL.'
pubDate: 'Jan 22 2024'
---

With the release of the App router in NextJS 13, we lost the ability to use `next/router` `shallow` property. Subsequently, we also lost the ability to use the shallow property to prevent a page from reloading when adding query parameters to the page URL.

In the past, we could use the shallow property in `router.push` and `router.replace` to include query parameters to the URL without any page flashing or re-rendering - an optimization that made applications feel smoother and faster. This functionality could be achieved with the following code:

```js
const router = useRouter();

useEffect(() => {
  router.push(`${router.asPath}?${qs.stringify(filters)}`, undefined, {
    shallow: true,
  });
}, [filters]);
```

However, with NextJS 13 and its App router, we need to use the browser's history API if we want to add query parameters to the page URL without causing a page refresh.

This allows us to use commands like `window.history.replace` and `window.history.push` to add query parameters to the page, achieving the same behavior without the need for a full page reload. We can achieve this with the following code:

```js
useEffect(() => {
  window.history.pushState(null, "", `${pathname}?${qs.stringify(filters)}`);
}, [filters]);
```

If you want to test this code out for yourself and learn more about the differences between the pages router and app router, you can try it out on StackBlitz using this URL: https://stackblitz.com/edit/stackblitz-starters-kukfbv