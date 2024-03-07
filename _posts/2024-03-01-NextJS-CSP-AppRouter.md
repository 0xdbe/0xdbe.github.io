---
layout: post
title: "Next.js: consequence of AppRouter on your CSP"
date: 20240301
published: true
categories: [Next.js]
tags: Security, AppSec, Next.js
canonical_url: https://0xdbe.github.io/NextJS-CSP-AppRouter/
---

With the integration of AppRouter, Next.js undergoes significant internal changes in component loading and management.
Underneath, AppRouter defaults to employing SSR (Server-Side Rendering) and leverages React Server Components (RSC).
However, this transition brings about consequential impacts on CSP (Content Security Policy).

> This article is written down as an ADR (Architectural Decision Record) because, from my point of view, this is an important security decision.
> This includes the context of how the decision was made and the consequences of adopting the decision.
> So, if you find it useful, you can share this ADR with your team members, and perhaps it will prove to be an effective strategy.

## Issue

All Server Components are loaded using an inline script as depicted below:

```html
<script>
  self.__next_f.push(
    [
        1,
        "1:HL[
            \"/_next/static/media/c9a5bc6a7c948fb0-s.p.woff2\",
            \"font\",{\"crossOrigin\":\"\",
            \"type\":\"font/woff2\"}
           ]\n
         2:HL[
            \"/_next/static/css/0a4b1961022ab8f5.css\",
            \"style\"
           ]\n
         0:\"$L3\"\n
        "
    ]
    )
</script>
```

This script invokes ``self.__next_f.push()`` with the RSC payload, consisting of serialized HTML content.
From a security standpoint, this presents an issue due to the presence of an unsafe inline script.

## Considered Options

### Disable AppRouter

With ``create-next-app``, it's always an option to create a Next.js app without AppRouter.
When prompted, select ``No`` to the following question:

```
Would you like to use App Router? (recommended)  No / Yes
```

This choice replaces the AppRouter with the old PageRouter.
However, it's worth noting that this is not recommended by Next.js, as the PageRouter may potentially be deprecated in future releases.

### Disable SSR

In order to disable Server Side Rendering, add ``use client`` in each component and layout.
Unfortunately, this means forfeiting the benefits of SSR.

### Use unsafe-inline

The simplest way to make AppRouter work is by adding ``unsafe-inline``.
However, this keyword disables protection against XSS (Cross-Site Scripting).

### Use nonce

Another option to allow inline script is to use a nonce.
A nonce is a unique, random string of characters created for one-time use.

This nonce must be inserted in the HTTP Header:


```
script-src: nonce-1234
```

And also in the HTML document:

```
<script nonce="1234">
  //...
</script>
```


## Decision

Among the considered options, the only safe solution is to use a CSP with nonce in the ``script-src`` directive.
While nonces should generally be avoided in a CSP, Next.js doesn't currently provide any secure alternatives, and nonces are the only solution to maintain a strict CSP.

## Consequence

In order to generate a random nonce for each request, a static storage solution like Bucket S3 can no longer be used to host a Next.js application.
Thus, an application runtime is now required, potentially increasing hosting costs.
Additionally, pages including a random nonce cannot be cached through a CDN (Content Delivery Network).


## Links

- [Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) from [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org)
- [The CSP unsafe-inline Source List Keyword](https://content-security-policy.com/unsafe-inline/)
- [Nonce attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce) from [MDN](https://developer.mozilla.org)
- [Configuring Content Security Policy](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy) from [nextjs.org](https://nextjs.org/)
- [NEXTJS 13: self.__next_f.push](https://github.com/vercel/next.js/discussions/42170)
- [React Server Components, Explained](https://giuseppegurgone.com/react-server-component-explained)
