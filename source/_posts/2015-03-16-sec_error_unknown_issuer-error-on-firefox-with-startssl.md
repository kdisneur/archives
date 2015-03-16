---
layout: post
title: Fix `sec_error_unknown_issuer` error on Firefox for startSSL
tags:
  - nginx
  - firefox
  - SSL
---
All my websites use free SSL certificates from [Start SSL](http://startssl.com/). The certificates are free for non
transaction usage. All my websites are just presentation or services but don't accept money so I can use them freely.

The basic usage is to copy your `crt` and `key` files to the server and have an `Nginx` configuration like:

```nginx
server {
  listen 80;
  server_name my.host.com;

  rewrite ^(.*)$ https://my.host.com$1 permanent;
}

server {
  listen 443;
  server_name my.host.com;

  ssl                 on;
  ssl_certificate     /etc/certs/my.host.com.crt;
  ssl_certificate_key /etc/certs/my.host.com.key;

  location ~ / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header Host $http_host;

    proxy_pass http://127.0.0.1:8080;
    proxy_redirect off;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

It will work well on Google Chrome and Safari but unfortunately not on Firefox.

You will have the following error:

> The certificate is not trusted because the issuer certificate is unknown.
> (Error code: sec_error_unknown_issuer)

In order to fix this issue on Firefox you will have to create an unified certificate containing your certificate and the
[StartSSL intermediate CA certificate](http://www.startssl.com/certs/sub.class1.server.ca.pem). If you have a higher
certificate you will have to replace the `class1` in the link above, by your certificate class number.

Then you will be able to create your **unified certificate**:

```bash
cat my.host.com.crt sub.class1.server.ca.pem > my.host.com.unified-crt
```

And, finally, you can replace the certificate on your server and in your nginx configuration to use your new unified
certificate.

Well done, you now have a working SSL website working on Google Chrome, Safari and Firefox.
