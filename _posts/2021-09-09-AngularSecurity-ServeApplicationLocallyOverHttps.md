---
layout: post
title: Angular Security - Serve application locally over HTTPS
date: 20210909
published: true
categories: [Angular]
tags: Security, AppSec, Angular
canonical_url:
---

When you develop an Angular application, you will come to a point where you need to serve it on localhost over HTTPS. This is often the case if you need to interact with an identity provider such as Facebook, OAuth0, ... And by the way, testing locally with HTTPS could be useful to detect mixed content issues that can break a production HTTPS website.

This article will walk you through setting Angular to use locally-trusted development certificate with ``mkcert``. This is an easy way to expose an application over HTTPS without security warnings and you don't worry too much about OpenSSL.
 

## Install mkcert

[mkcert](https://github.com/FiloSottile/mkcert) is a simple tool for making locally-trusted development certificates.
This tool is written by Filippo Valsorda, Cryptographer and Go security leader. Follow instructions on https://github.com/FiloSottile/mkcert#installation to install this tool.


## Create local certificate authority

Run this command to create a new local certificate authority (CA):

```
mkcert -install
```

This command did the following:

- Generate CA certificate and its key, and store them in an application data folder in the user home, such as ``~/.local/share/mkcert/``
- Add this new certificate authority in trust stores (system, Firefox, Chrome, ...). So, all certificates issue with this CA will be trusted by your browser

Be aware that the ``rootCA-key.pem`` file that ``mkcert`` automatically generates gives complete power to intercept secure requests from your machine. So, do not share it to stay safe against MITM (Man-in-the-middle) attack.


## Get certificate

Run the following commands from Angular project directory:

```
mkdir tls
mkcert \
      -cert-file ./tls/localhost-cert.pem \
      -key-file ./tls/localhost-key.pem \
      -ecdsa \
      localhost 127.0.0.1 ::1
```

This certificate will expire in 3 months. In order to make renewal easier, we can create a shortcut for this command in ``package.json``:

```
"scripts": {
  "start": "npm run cert & ng serve",
  "cert": "mkdir -p ./tls & mkcert -cert-file ./tls/localhost-cert.pem -key-file ./tls/localhost-key.pem -ecdsa localhost 127.0.0.1 ::1"
}
```

Thus, you get a new certificate each time you start your application

Don't forget to add ``tls/*``in ``.gitignore`` to prevent publication of your private key.



## Serve application

In order to serve Angular application securely, add ``ssl``, ``sslCert`` and ``sslKey`` options to ``serve`` command:

```
ng serve \
      --ssl=true \
      --sslCert=./tls/localhost-cert.pem \
      --sslKey=./tls/localhost-key.pem
```

To avoid to write these options each time that we want to run this app over HTTPS, we can set them in the ``angular.json`` file:

```
"serve": {
          "builder": "@angular-devkit/build-angular:dev-server",
          "options": {
            "ssl": true,
            "sslCert": "tls/localhost-cert.pem",
            "sslKey": "tls/localhost-key.pem"
          }
}
```

Like so, you can easily serve your app by using ``npm start`` and then, browse [https://localhost:4200/](https://localhost:4200/) without security warning. But keep in mind that this could be used only for local development, not for production.
