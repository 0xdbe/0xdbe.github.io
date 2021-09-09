---
layout: post
title: Angular Security - Disable Inline Critical CSS
date: 20210905
published: true
categories: [Angular]
tags: Security, AppSec, Angular
canonical_url: https://0xdbe.github.io/AngularSecurity-DisableInlineCriticalCSS/
---

Improving load time is crucial for the success of your application. One way to reduce this load time is to optimize the CSS loading but it is quite tricky, because CSS files are render-blocking. This means that the browser must download and parse these files before starting to render the web page.

That's why Angular provides CSS optimization in order to reduce this render-blocking delay, and at the same time, to improve the First Contentful Paint (FCP). This optimization involves first inlining critical CSS and delaying the loading of non-critical CSS.

This article describes what's wrong with this optimization and how to disable it to keep a strict CSP (Content Security Policy).

## What's wrong?

Inline Critical CSS is an optimization that impacts our CSP (Content Security Policy):

```
style-src-elem 'unsafe-inline';    // For Inlining critical CSS
script-src     'unsafe-inline';    // For Delaying non-critical CSS
```

To understand why it is necessary, let's take a look at these practices.

### Inlining critical CSS

During the build process, Angular extracts first all CSS resources that block the rendering. Once critical CSS are extracted, Angular inlines them directly in the ``index.html`` file. In order to authorize inline CSS, we have to add the following content in our CSP:

```
style-src-elem 'unsafe-inline';
```

With this configuration, our CSP is not able to block [CSS injections](https://www.netsparker.com/blog/web-security/private-data-stolen-exploiting-css-injection/) anymore. This problem is not new since inline CSS is used by Angular for Component Style. So, inlining critical CSS should not further affect our CSP.

### Delaying non-critical CSS

After inlining critical CSS, the rest can be postponed. However, HTML and CSS don't support asynchronous loading for CSS files. To circumvent this issue, there is an Angular trick to load non-critical CSS asynchronously using ``media`` attribute:

```html
<link rel="stylesheet"
  href="styles.1d6c8a3b8017c43eaeda.css"
  media="print"
  onload="this.media='all'">
```

Media type (``print``) doesn’t match the current environment, so the browser decides that it is less important and loads the stylesheet asynchronously, without delaying the page rendering. On load, we change media type so that the stylesheet gets applied to screens. In order to authorize event handlers that run inline script, we have to include the following content in our CSP:

```
script-src 'unsafe-inline';
```

This configuration almost defeats the purpose of CSP, thus we could be exposed to XSS attacks.


## How to fix this?

For security purposes, inline critical CSS must be disabled to keep a strict CSP.

Inline critical CSS is a new optimization introduced in Angular 11.1. However it was disabled by default. This optimization is now enabled by default in v12 and you have to set ``inlineCritical`` to ``false`` in ``angular.json`` for each configuration:

```json
{
  "configurations": {
    "production": {
      "optimization": {
        "scripts": true,
        "styles": {
          "minify": true,
          "inlineCritical": false
        },
        "fonts": false
      }
    }
  }
}
```

With this configuration, Angular includes CSS as before:

```html
<link rel="stylesheet" href="styles.css">
```

And we don't have to weaken our CSP!

