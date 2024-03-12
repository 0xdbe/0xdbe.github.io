---
layout: post
title: "Next.js: consequence of Next/Image on your CSP"
date: 20240312
published: true
categories: [Next.js]
tags: Security, AppSec, Next.js
canonical_url: https://0xdbe.github.io/NextJS-CSP-NextImage/
---

The ``Next/Image`` component is a crucial part of the Next.js framework, offering image optimization functionalities.
However, utilizing this component can have significant implications for Content Security Policy (CSP).

> This article is documented as an ADR (Architectural Decision Record) because, in my perspective, this represents a crucial security decision.
> This includes the context of how the decision was made and the consequences of adopting it
> Feel free to share this ADR with your team members if you find it beneficial.

## Issue

To understand the implications of ``Next/Image``, let's examine the following example extracted from the default Next.js template:

``` html
<Image
  className="relative dark:drop-shadow-[0_0_0.3rem_#ffffff70] dark:invert"
  src="/next.svg"
  alt="Next.js Logo"
/>
```

This component generates the following HTML code:

```html
<img
  alt="Next.js Logo"
  style="color:transparent"
  src="/next.svg"
>
```

As observed, the outcome contains an unsafe inline style.
In this scenario, ``Next/Image`` adds an inline style to conceal the ``alt`` text.

``Next/Image`` provides many other properties to configure an image.
Some of them are also translated into inline style, such as ``fill`` property:

```html
<img
  style="position:absolute;height:100%;width:100%;left:0;top:0;right:0;bottom:0;object-fit:contain;color:transparent"
>
```

## Considered Options

### Unsafe Inline

The simplest approach is to allow the ``unsafe-inline`` keyword for ``style-src`` directive.
However, this leaves you vulnerable to CSS injection attacks.

### Unsafe Hashes

Since styles for images are predetermined, we can pre-compute hashes for styles:

```
echo -n "color:transparent" | openssl dgst -sha256 -binary | openssl base64 -A
```

Subsequently, you can append the following content to your CSP:

```
style-src 'self' 'sha256-zlqnbDt84zf1iSefLU/ImC54isoprH/MRiVZGskwexk=' 'unsafe-hashes'
```

Unfortunately, the ``unsafe-hashes`` keyword is necessary to permit hashes for the ``style`` attribute.

### Remove Style

Even without utilizing any properties that generate inline style, there's only one persistent style attribute that you can't eliminate: ``color:transparent``.

To remove the style attribute, it is possible to create a new component called ``ImageNoStyle``, which omits the style attribute:

``` typescript
// ImageNoStyle.tsx
import NextImage, { getImageProps } from 'next/image';
import { ComponentProps } from 'react';

export default function ImageNoStyle(props: ComponentProps<typeof NextImage>) {
  const { props: nextProps } = getImageProps({
    ...props,
  });

  const { style: _omit, ...delegated } = nextProps;

  return <img {...delegated} />;
}
```

This component uses ``getImageProps()``, available from Next.js 14.1, to generate image properties and omit the style attribute.
Then, ``ImageNoStyle`` can be used like this with the same properties as ``NextImage``:

``` html
<ImageNoStyle
  className="relative dark:drop-shadow-[0_0_0.3rem_#ffffff70] dark:invert"
  src="/next.svg"
  alt="Next.js Logo"
  width={180}
  height={37}
  priority
/>
```

However, it's worth noting that this workaround removes inline style for images.
Consequently, this workaround can modify rendering of an existing website.

### HTML Tag

The basic HTML ``<img>`` tag can be used to embed an image in react component.

## Decision

Pending a resolution for issues [61388](https://github.com/vercel/next.js/issues/61388) and [45184](https://github.com/vercel/next.js/issues/45184)), it's advisable to refrain from using ``Next/Image``.
Employing the ``<img>`` HTML tag helps maintain a strict CSP (Content Security Policy).

> This decision is relevant to a new application.
> For existing applications, the choice may differ, particularly if image optimization is already in use.

## Consequence

Opting for the basic ``<img>`` HTML tag means foregoing the image optimization features provided by Next.js, as outlined in the [image optimization](https://nextjs.org/docs/pages/building-your-application/optimizing/images) documentation.

## Links

- [inline style color:transparent in Image #61209](https://github.com/vercel/next.js/discussions/61209) on GitHub
- [CSP error when using next/image #45184](https://github.com/vercel/next.js/issues/45184) on GitHub
- [next/Image uses unsafe inline style #61388](https://github.com/vercel/next.js/issues/61388) on GitHub
- [Image Optimization](https://nextjs.org/docs/pages/building-your-application/optimizing/images)
