---
layout: post
title:  "Compressing JSON Nginx Responses"
tags: ["compression", "json", "nginx"]
---

Simply including `gzip on;` in your [Nginx](http://nginx.org/) configuration only compresses responses with the MIME type `text/html`. If Nginx is returning JSON data with a MIME type `application/json`, you can enable compression for these responses by doing:

```text
http {
  ...
  gzip             on;
  gzip_comp_level  9;
  gzip_types       application/json;
  ...
}
```

This is especially important for mobile clients that may download such responses over slow networks.

Afterward, don't forget to actually examine the HTTP response to ensure that Nginx is compressing its body. If you don't have access to its raw form, inspect the HTTP headers of the response instead. You should should find the following two lines:

```text
Content-Type: application/json
Content-Encoding: gzip
```

For more information, see the [gzip module documentation](http://nginx.org/en/docs/http/ngx_http_gzip_module.html).

