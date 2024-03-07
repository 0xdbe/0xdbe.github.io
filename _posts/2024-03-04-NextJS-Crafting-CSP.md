---
layout: post
title: "Next.js: Crafting a Strict CSP"
date: 20240304
published: true
categories: [Next.js]
tags: Security, AppSec, Next.js
canonical_url: https://0xdbe.github.io/NextJS-Crafting-CSP/
---

Next.js lacks many built-in security measures.
In fact, it doesn't offer predefined configurations for your Content Security Policy (CSP).
Consequently, setting up CSP becomes your responsibility.
Let's explore how we can implement a CSP.

## Requirements

To ensure a Content Security Policy that aligns with the current standards, we need to meet the following requirements:

- must include a **nonce** because Next.js often utilizes inline scripts.
- must be delivered through an HTTP Header. Avoid embedding a CSP in a ``meta`` tag with ``http-equiv`` as many unsuported features, such as violation reports and frame-ancestors directive.

## Middleware

In Next.js, middleware allows you to execute a function before a request is completed.
In our case, a middleware will be used to generate a random nonce and then create HTTP headers before the page renders.

Here is the middleware function, which can be adjusted if a middleware chain is used:

```typescript
// file: middleware.ts
export function middleware(request: NextRequest) {

  // step 1
  const response = initResponse()

  // step 2
  const reportUri = getReportUri()
  response.headers.set(
    'Report-To', getReportToHeaderValue(reportUri)
  )

  // step 3
  const nonce = getNonce()

  // step 4
  response.headers.set(
    'Content-Security-Policy',
    getContentSecurityPolicyHeaderValue(nonce, reportUri)
  )

  return response
}
```

## Helper functions

### Step 1: Initialize Response

The first step involves initializing the response using the following function:

```typescript
function initResponse(): NextResponse {
  return NextResponse.next()
}
```

Using ``NextResponse.next()`` is suitable, in this case, to create an empty response.
Although if you're using a chain middleware where the response is already initialized, you might skip this step.

### Step 2: Define CSP Violation Reporting

The second step is defining where CSP violation reports will be sent.

Here's the function:

```typescript
function getReportToHeaderValue(reportUri: string): string {
  const reportTo = {
    group: 'csp',
    max_age: 10886400, //1 day
    endpoints: [{ url: reportUri }],
  }
  return JSON.stringify(reportTo)
}
```

Certain providers like Sentry or Datadog offer the capability to collect information on CSP (Content-Security-Policy) violations. Remember, if you're testing violation reports, they won't be sent if you're using ``localhost`` hostname or an insecure protocol like ``http``.

### Step 3: Generate Nonce

The third step involves generating a random nonce using ``randomUUID()``:

```typescript
function getNonce(): string {
  return Buffer.from(crypto.randomUUID()).toString('base64')
}
```

### Step 4: Generate CSP Content

The final step is to generate the content of the CSP, which is perhaps the most intricate task.

Here's a function to define the CSP:


```typescript
function getContentSecurityPolicyHeaderValue(
    nonce: string,
    reportUri: string,
  ): string {

    // Default CSP for Next.js
    const contentSecurityPolicyDirective = {
      'base-uri': [`'self'`],
      'default-src': [`'none'`],
      'frame-ancestors': [`'none'`],
      'font-src': [`'self'`],
      'form-action': [`'self'`],
      'frame-src': [`'self'`],
      'connect-src': [`'self'`],
      'img-src': [`'self'`],
      'manifest-src': [`'self'`],
      'object-src': [`'none'`],
      'report-uri': [reportUri], // for old browsers like Firefox
      'report-to': ['csp'], // for modern browsers like Chrome
      'script-src': [
        `'nonce-${nonce}'`,
        `'strict-dynamic'`, // force hashes and nonces over domain host lists
      ],
      'style-src': [`'self'`],
    }

    if (process.env.NODE_ENV === 'development') {
      // Webpack use eval() in development mode for automatic JS reloading
      contentSecurityPolicyDirective['script-src'].push(`'unsafe-eval'`)
    }

    if (process.env.NEXT_PUBLIC_VERCEL_ENV === 'preview') {
      contentSecurityPolicyDirective['connect-src'].push('https://vercel.live')
      contentSecurityPolicyDirective['connect-src'].push('wss://*.pusher.com')
      contentSecurityPolicyDirective['img-src'].push('https://vercel.com')
      contentSecurityPolicyDirective['font-src'].push('https://vercel.live')
      contentSecurityPolicyDirective['frame-src'].push('https://vercel.live')
      contentSecurityPolicyDirective['style-src'].push('https://vercel.live')
    }

    return Object.entries(contentSecurityPolicyDirective)
      .map(([key, value]) => `${key} ${value.join(' ')}`)
      .join('; ')

}
```

If you need to complete the default CSP, use the push method to add additional sources.

## Page Render

Once we have middleware that creates a CSP with a nonce, we need to include this nonce on pages.
Every time a page is viewed, a fresh nonce should be generated.
This means you must use dynamic rendering to add nonces:

```
// file: app/layout.tsx
export const dynamic = 'force-dynamic'
```

With this option, Next.js will be able to insert a random number on the rendered page.

To achieve this, Next.js utilizes [getScriptNonceFromHeader](https://github.com/vercel/next.js/blob/v14.1.0/packages/next/src/server/app-render/get-script-nonce-from-header.tsx) to extract the nonce from the CSP HTTP header.
Then, [AppRender](https://github.com/vercel/next.js/blob/7744cc91beae7169c264e4a7508ef55256c4ab42/packages/next/src/server/app-render/app-render.tsx) includes the nonce in all script elements.

**Warning:** Keep in mind that dynamic rendering can increase hosting costs. For example, on Vercel, page rendering is done as a serverless function.

## Conclusion

Next.js is not designed to use a strict Content-Security-Policy, which may discourage many software engineers.
I hope this article can help some of them who might be tempted to use the ``unsafe-inline`` keyword.

If you haven't yet selected a framework for your next secure app, consider avoiding Next.js and opting for a better-suited one, although I haven't identified it yet.

## Link

- [Configuring Content Security Policy](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy) from [nextjs.org](https://nextjs.org/)
- [Example a CSP header with a meta tag](https://content-security-policy.com/examples/meta/) from [CSP Reference Guide](https://content-security-policy.com/)
