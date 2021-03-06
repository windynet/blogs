# 前后端分离实践总结

### 什么时候需要分离

一般来说，工具网站、内部网站等，例如邮箱，没有 SEO 要求，完全可以做前后端分离，可以大幅提高程序可维护性。

一般来说，内容型网站，例如博客、新闻、问答、讨论等，有 SEO 的硬性要求，可以考虑对正文、标题、答案等内容在服务端进行渲染，其它内容则完全可以在异步获得数据后，在前端进行渲染。

内容型网站在访问频率很高时，可以把页面以静态文件的形式静态化，根据具体场景，定时渲染，更新生成的静态文件。

### 主要效果

1. 后端只提供 API，独立的代码，独立部署，可能会有多个域名，例如 https://example.com/api/ 和 https://api.example.com/
2. 前端自己渲染，提供 html/js/css 等，独立的代码，独立部署，通过 https://example.com/ 访问

### 前端 web server 用 nginx

这是一个常见的方案，如果前端文件在 `/opt/static`，则 nginx 的配置文件类似：

```nginx
    server {
        listen       80;
        listen       [::]:80;
        server_name example.com www.example.com;

        location ~*\.(js|css|jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|map|mp4|ogg|ogv|webm|htc|json|ttf|woff)$ {
            root         /opt/static;
            expires 1M;
            access_log off;
            add_header Cache-Control public;
        }
        location ~*\.html$ {
            root         /opt/static;
            index index.html;
            access_log off;
        }
    }
```

这里 html 和其它文件不同在于：Cache-Control 是不一样的，html 文件一般不会加版本控制，更新时会改变文件的更新时间，所以可以采用默认策略；而其它文件，一般是在文件名上加版本控制，或者创建后就不再变化，这样使用 public 要更好一些。

### 配置 https 和 http2.0

https 正在普及，成本也不高。如果 crt 和 key 文件分别放在 `/opt/1.crt` 和 `/opt/1.key`，配置会变成：

```nginx
    server {
        # changes start
        listen       443 ssl http2;
        listen       [::]:443;
        ssl on;
        ssl_certificate /opt/1.crt;
        ssl_certificate_key /opt/1.key;
        # changes end
        server_name example.com www.example.com;

        location ~*\.(js|css|jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|map|mp4|ogg|ogv|webm|htc|json|ttf|woff)$ {
            root         /opt/static;
            expires 1M;
            access_log off;
            add_header Cache-Control public;
        }
        location ~*\.html$ {
            root         /opt/static;
            index index.html;
            access_log off;
        }
    }
```

其中 nginx 在 1.9.5 之后才支持 http2.0，如果 nginx 版本不够，要去掉 `http2`。

### 加上后端

后端需要自己监听一个端口，以 node 为例，可以以 pm2/forever 运行，监听 localhost:3000，这时候就需要 nginx 反代这个服务：

```nginx
    # changes start
    upstream backend {
        server localhost:3000;
    }
    # changes end
    server {
        listen       443 ssl http2;
        listen       [::]:443;
        ssl on;
        ssl_certificate /opt/1.crt;
        ssl_certificate_key /opt/1.key;
        server_name example.com www.example.com;

        location ~*\.(js|css|jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|map|mp4|ogg|ogv|webm|htc|json|ttf|woff)$ {
            root         /opt/static;
            expires 1M;
            access_log off;
            add_header Cache-Control public;
        }
        location ~*\.html$ {
            root         /opt/static;
            index index.html;
            access_log off;
        }
        # changes start
        location / {
            proxy_pass http://backend;
        }
        # changes end
    }
```

这时候目标都已完成。

### 如果使用了 socket.io

nginx 是支持 websocket 的，可以观察到 socket.io 的通信 url 形式是 `/socket.io/*`。这时候的配置：

```nginx
    upstream backend {
        server localhost:3000;
    }
    # changes start
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }
    # changes end
    server {
        listen       443 ssl http2;
        listen       [::]:443;
        ssl on;
        ssl_certificate /opt/1.crt;
        ssl_certificate_key /opt/1.key;
        server_name example.com www.example.com;

        # changes start
        location ~*/socket.io/* {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
        }
        # changes end
        location ~*\.(js|css|jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|map|mp4|ogg|ogv|webm|htc|json|ttf|woff)$ {
            root         /opt/static;
            expires 1M;
            access_log off;
            add_header Cache-Control public;
        }
        location ~*\.html$ {
            root         /opt/static;
            index index.html;
            access_log off;
        }
        location / {
            proxy_pass http://backend;
        }
    }
```

### 跨域问题

主要有两种解决方案，jsonp 和 cors，jsonp 本质上是 get 一个 js script，优点是浏览器支持度好，缺点是只支持 GET，而 cors 可以支持 GET/POST/PUT/DELETE 等等，但 IE6 不支持 cors。

跨域问题主要靠后端来处理，以 node 为例，可以使用库 https://www.npmjs.com/package/cors ，把支持跨域的域名加入配置就好。

如果跨域，请求是默认不带 cookie 的，如果需要，可以在前端设置，以 jquery 为例：

```javascript
$.ajaxSetup({
    xhrFields: {
        withCredentials: true
    }
});
```

### 前端开发环境的跨域问题

主要有三种解决方案，改 host、使用 fiddler script 修改 host 和使用本地 nginx，以后者为例，如果通过本地 3000 端口访问到的是 https://example.com ：

```nginx
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }
    server {
        listen       3000;
        server_name  localhost;

        location ~*/socket.io/* {
            proxy_pass https://example.com;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
        }

        location / {
            add_header Access-Control-Allow-Origin http://localhost:8000;
            proxy_pass https://example.com;
        }
    }
```

这里可以支持 websocket，其中 8000 是前端页面的监听端口。

### 如果需要加入多个 web server

强大的 nginx 就可以做到，还可以做负载均衡。

配置类似于：

```nginx
    upstream backend {
        server localhost:3000 weight=2;
        server localhost:3001 weight=1;
    }
```

### 如果要规避 nginx 单点的风险

为 DNS 解析增加多个 A 纪录，根据运营商的不同，解析到不同的 IP

### 如果前端是单页应用

这种情况，一般是前端切换路由时，URL 变化了，如果没有特殊地处理，页面被刷新后，会出现 404。

以 react-router 为例，同时假设前端路由都以 `.html` 结尾，nginx 配置变成：

```nginx
    upstream backend {
        server localhost:3000;
    }
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }
    server {
        listen       443 ssl http2;
        listen       [::]:443;
        ssl on;
        ssl_certificate /opt/1.crt;
        ssl_certificate_key /opt/1.key;
        server_name example.com www.example.com;

        location ~*/socket.io/* {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
        }
        location ~*\.(js|css|jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|map|mp4|ogg|ogv|webm|htc|json|ttf|woff)$ {
            root         /opt/static;
            expires 1M;
            access_log off;
            add_header Cache-Control public;
        }
        # changes start
        location ~*\.html$ {
            root         /opt/static;
            try_files $uri /index.html;
            access_log off;
        }
        location =/ {
            root         /opt/static;
            index index.html;
            access_log off;
        }
        # changes end
        location / {
            proxy_pass http://backend;
        }
    }
```

这里使用 try_files，对于以 `.html` 结尾的路由，如果找不到文件，会返回 `index.html`，react-router 会根据当前 URL 路由到正确的组件。

### 如果后端需要获得客户端真实 IP

这种情况，一般是后端需要根据真实 IP，来限制 API 访问频率。

nginx 配置变成：

```nginx
    upstream backend {
        server localhost:3000;
    }
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }
    server {
        listen       443 ssl http2;
        listen       [::]:443;
        ssl on;
        ssl_certificate /opt/1.crt;
        ssl_certificate_key /opt/1.key;
        server_name example.com www.example.com;

        location ~*/socket.io/* {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
        }
        location ~*\.(js|css|jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|map|mp4|ogg|ogv|webm|htc|json|ttf|woff)$ {
            root         /opt/static;
            expires 1M;
            access_log off;
            add_header Cache-Control public;
        }
        location ~*\.html$ {
            root         /opt/static;
            try_files $uri /index.html;
            access_log off;
        }
        location =/ {
            root         /opt/static;
            index index.html;
            access_log off;
        }
        location / {
            proxy_pass http://backend;
            # changes start
            proxy_set_header        Host            $host;
            proxy_set_header        X-Real-IP       $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            # changes end
        }
    }
```

这时，客户端真实 IP 存在于名为 `X-Real-IP` 或 `X-Forwarded-For` 的 header 中
