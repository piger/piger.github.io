---
title: "NGINX, X-Forwarded-For and the realip module"
date: 2022-04-10T12:53:20+01:00
tags:
- nginx
- lua
summary: "How to send the correct X-Forwarded-For header while using the realip module"
toc: false
---

There's a bug in NGINX that affects how the `X-Forwarded-For` header is populated when the
[ngx_http_realip_module](http://nginx.org/en/docs/http/ngx_http_realip_module.html) is used;
the bug has been there for a long time and doesn't look like it's going to be fixed
anytime soon, by the look of [this](https://trac.nginx.org/nginx/ticket/2127) bug report.

The bug manifests itself when you have two or more proxies in front of your backend, for example
when you use [Cloudflare](https://www.cloudflare.com/); say for example you have proxy1
(10.20.30.2) and proxy2 (10.20.30.3) having the following configuration:

proxy1 (10.20.30.2)

```nginx
http {
    server {
        server_name _;

        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Real-IP       $remote_addr;
            proxy_pass http://10.20.30.3;
        }
    }
}
```

proxy2 (10.20.30.3)

```nginx
http {
    set_real_ip_from 10.20.30.2/32;
    real_ip_recursive on;
    real_ip_header X-Forwarded-For;

    server {
        server_name _;

        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Real-IP       $remote_addr;
            proxy_pass http://backend;
        }
    }
}
```

and you serve a request for a client (1.2.3.4); you would expect that the backend would receive
the following headers:

```
X-Forwarded-For: 1.2.3.4, 10.20.30.2
X-Real-IP: 1.2.3.4
```

But in reality it will receive these:

```
X-Forwarded-For: 1.2.3.4, 1.2.3.4
X-Real-IP: 1.2.3.4
```

I think this happens because by the time NGINX give a value to `$proxy_add_x_forwarded_for` it
has already replaced `$remote_addr` with the original client's IP.

One solution I've found is to use a tiny bit of Lua to create the _correct_ version of
`X-Forwarded-For` in the intermediate proxy (proxy2) and execute it in the right phase; in my
experiments I've found that the "access" phase is the only one that works correctly.

The configuration for proxy2 now looks like this:

```nginx
http {
    set_real_ip_from 10.20.30.2/32;
    real_ip_recursive on;
    real_ip_header X-Forwarded-For;

    map $request $realip_add_x_forwarded_for { default ""; }
    access_by_lua_block {
        require("realip-x-forwarded-for").run()
    }

    server {
        server_name _;

        location / {
            proxy_set_header X-Forwarded-For $realip_add_x_forwarded_for;
            proxy_set_header X-Real-IP       $remote_addr;
            proxy_pass http://backend;
        }
    }
}
```

And the Lua script:

```lua
local _M = {}

function _M.run()
    if (ngx.var.http_x_forwarded_for == "" or ngx.var.http_x_forwarded_for == nil) then
        ngx.var.realip_add_x_forwarded_for = ngx.var.realip_remote_addr
    else
        ngx.var.realip_add_x_forwarded_for = ngx.var.http_x_forwarded_for .. ", " .. ngx.var.realip_remote_addr
    end
end

return _M
```

Note how we have to give a _default_ value to `$realip_add_x_forwarded_for` using a
[map](http://nginx.org/en/docs/http/ngx_http_map_module.html), because nginx won't allow `set`
statements in the `http` level.

Demo:

```
$ curl http://proxy1/
-> X-Forwarded-For: 1.2.3.4, 10.20.30.2
-> X-Real-Ip: 1.2.3.4

$ curl http://proxy2/
-> X-Forwarded-For: 1.2.3.4
-> X-Real-Ip: 1.2.3.4

$ curl -H "X-Forwarded-For: 8.8.8.8" http://proxy1
-> X-Forwarded-For: 8.8.8.8, 1.2.3.4, 10.20.30.2
-> X-Real-Ip: 1.2.3.4

$ curl -H "X-Forwarded-For: 8.8.8.8" -H "X-Forwarded-For: 9.9.9.9" http://proxy1
-> X-Forwarded-For: 8.8.8.8, 9.9.9.9, 1.2.3.4, 10.20.30.2
-> X-Real-Ip: 1.2.3.4
```

You can experiment with this setup yourself with my [nginx-lab](https://github.com/piger/nginx-lab)
if you checkout [this PR](https://github.com/piger/nginx-lab/pull/1/files).
