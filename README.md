# Nginx Configuration Snippets
A collection of useful Nginx configuration snippets inspired by
[.htaccess snippets](https://github.com/phanan/htaccess).

## Table of Contents
- [The Nginx Command](#the-nginx-command)
- [Rewrite and Redirection](#rewrite-and-redirection)
    - [Force www](#force-www)
    - [Force no-www](#force-no-www)
    - [Force HTTPS](#force-https)
    - [Force Trailing Slash](#force-trailing-slash)
    - [Redirect a Single Page](#redirect-a-single-page)
    - [Redirect an Entire Site](#redirect-an-entire-site)
    - [Redirect an Entire Sub Path](#redirect-an-entire-sub-path)
- [Performance](#performance)
    - [Contents Caching](#contents-caching)
    - [Gzip Compression](#gzip-compression)
    - [Open File Cache](#open-file-cache)
    - [SSL Cache](#ssl-cache)
    - [Upstream Keepalive](#upstream-keepalive)
- [Monitoring](#monitoring)
- [Security](#security)
    - [Enable Basic Authentication](#enable-basic-authentication)
    - [Only Allow Access From Localhost](#only-allow-access-from-localhost)
    - [Secure SSL settings](#secure-ssl-settings)
- [Miscellaneous](#miscellaneous)
    - [Sub-Request Upon Completion](#sub-request-upon-completion)
    - [Enable Cross Origin Resource Sharing](#enable-cross-origin-resource-sharing)
- [Links](#links)

## The Nginx Command
The `nginx` command can be used to perform some useful actions when Nginx is running.

- Get current Nginx version and its configured compiling parameters: `nginx -V`
- Test the current Nginx configuration file and / or check its location: `nginx -t`
- Reload the configuration without restarting Nginx: `nginx -s reload`


## Rewrite and Redirection

### Force www
The [right way](http://nginx.org/en/docs/http/converting_rewrite_rules.html)
is to define a separated server for the naked domain and redirect it.
```nginx
server {
    listen 80;
    server_name example.org;
    return 301 $scheme://www.example.org$request_uri;
}

server {
    listen 80;
    server_name www.example.org;
    ...
}
```

Note that this also works with HTTPS site.

### Force no-www
Again, the right way is to define a separated server for the www domain and redirect it.
```nginx
server {
    listen 80;
    server_name example.org;
}

server {
    listen 80;
    server_name www.example.org;
    return 301 $scheme://example.org$request_uri;
}
```

### Force HTTPS
This is also handled by the 2 server blocks approach.
```nginx
server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;

    # let the browsers know that we only accept HTTPS
    add_header Strict-Transport-Security max-age=2592000;

    ...
}
```

### Force Trailing Slash
This configuration only add trailing slash to URL that does not contain a dot because you probably don't want to add that trailing slash to your static files.
[Source](http://stackoverflow.com/questions/645853/add-slash-to-the-end-of-every-url-need-rewrite-rule-for-nginx).
```nginx
rewrite ^([^.]*[^/])$ $1/ permanent;
```

### Redirect a Single Page
```nginx
server {
    location = /oldpage.html {
        return 301 http://example.org/newpage.html;
    }
}
```

### Redirect an Entire Site
```nginx
server {
    server_name old-site.com
    return 301 $scheme://new-site.com$request_uri;
}
```

### Redirect an Entire Sub Path
```nginx
location /old-site {
    rewrite ^/old-site/(.*) http://example.org/new-site/$1 permanent;
}
```


## Performance

### Contents Caching
Allow browsers to cache your static contents for basically forever. Nginx will set both `Expires` and `Cache-Control` header for you.
```nginx
location /static {
    root /data;
    expires max;
}
```

If you want to ask the browsers to **never** cache the response (e.g. for tracking requests), use `-1`.
```nginx
location = /empty.gif {
    empty_gif;
    expires -1;
}
```

### Gzip Compression
```nginx
gzip  on;
gzip_buffers 16 8k;
gzip_comp_level 6;
gzip_http_version 1.1;
gzip_min_length 256;
gzip_proxied any;
gzip_vary on;
gzip_types
    text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml
    text/javascript application/javascript application/x-javascript
    text/x-json application/json application/x-web-app-manifest+json
    text/css text/plain text/x-component
    font/opentype application/x-font-ttf application/vnd.ms-fontobject
    image/x-icon;
gzip_disable  "msie6";
```

### Open File Cache
If you have _a lot_ of static files to serve through Nginx then caching of the files' metadata (not the actual files' contents) can save some latency.
```nginx
open_file_cache max=1000 inactive=20s;
open_file_cache_valid 30s;
open_file_cache_min_uses 2;
open_file_cache_errors on;
```

### SSL Cache
Enable SSL cache for SSL sessions resumption, so that sub sequent SSL/TLS connection handshakes can be shortened and reduce total SSL overhead.
```nginx
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
```

### Upstream Keepalive
Enable the upstream connection cache for better reuse of connections to upstream servers. [Source](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive).
```nginx
upstream backend {
    server 127.0.0.1:8080;
    keepalive 32;
}

server {
    ...
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```


## Monitoring

The [Stub Status](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html), which is not built by default, is a very simple to setup module but only provide basic status of Nginx.
```nginx
location /status {
    stub_status on;
    access_log off;
}
```

It provides the following status for the whole Nginx server in plain text(!) format:
- Client connections: accepted, handled, active (includes reading, writing and waiting).
- Total number of client requests.

**[Shameless Plug]** A _better_ way to capture Nginx status can be added by using [Luameter](https://luameter.com) which is a bit more complicated to setup and required the Nginx Lua module (which is awesome). It provides following metrics for each [configurable group](https://luameter.com/configuration) as a JSON API:
- Total number of requests / responses.
- Total number of responses groupped by status code: 1xx, 2xx, 3xx, 4xx, 5xx.
- Total bytes received from / sent to client.
- Sampled latency snapshot for estimation of: mean, max, median, 99th percentile, etc., latency.
- Moving average rate of requests for easier monitoring and predicting.
- And [some more](https://luameter.com/metrics).

[Here is a sample dashboard built with Luameter's metrics](https://luameter.com/demo).

[ngxtop](https://github.com/lebinh/ngxtop) is also a good way to check for Nginx status and checking / troubleshooting a live server.


## Security

### Enable Basic Authentication
You will need a user password file somewhere first.
```
name:{PLAIN}plain-text-password
```

Then add below config to `server`/`location` block that need to be protected.
```nginx
auth_basic "This is Protected";
auth_basic_user_file /path/to/password-file;
```

### Only Allow Access From Localhost
```nginx
location /local {
    allow 127.0.0.1;
    deny all;
    ...
}
```

### Secure SSL settings
- Disable SSLv3 which is enabled by default. This prevents [POODLE SSL Attack](http://nginx.com/blog/nginx-poodle-ssl/).
- Ciphers that best allow protection from Beast. [Mozilla Server Side TLS and Nginx]( https://wiki.mozilla.org/Security/Server_Side_TLS#Nginx)
```nginx
# don’t use SSLv3 ref: POODLE CVE-2014-356 - http://nginx.com/blog/nginx-poodle-ssl/
ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;  

# Ciphers set to best allow protection from Beast, while providing forwarding secrecy, as defined by Mozilla (Intermediate Set) - https://wiki.mozilla.org/Security/Server_Side_TLS#Nginx
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
ssl_prefer_server_ciphers  on;

```

## Miscellaneous

### Sub-Request Upon Completion
There are some cases that you want to pass the request to another backend _in addition to and after_ serving it. One use case is to track the number of completed downloads by calling an API after user completed download a file. Another use case is for tracking request where you want to return as fast as possible (perhaps with an `empty_gif`) and then do the actual recording in background. The [post_action](http://wiki.nginx.org/HttpCoreModule#post_action) that allows you to define a sub-request that will be fired upon completion of the current request are [perfect solution](http://mailman.nginx.org/pipermail/nginx/2008-April/004524.html) for these use cases.
```nginx
location = /empty.gif {
    empty_gif;
    expires -1;
    post_action @track; 
}

location @track {
    internal;
    proxy_pass http://tracking-backend;
}
```

### Enable Cross Origin Resource Sharing
Simple, wide-open configuration to allow cross-domain requests to your server.
```nginx
location ~* \.(eot|ttf|woff) {
    add_header Access-Control-Allow-Origin *;
}
```


## Links
Some other awesome resources for configuring Nginx:

- [Nginx Official Guide](http://nginx.com/resources/admin-guide/)
- [HTML 5 Boilerplate's Sample Nginx Configuration](https://github.com/h5bp/server-configs-nginx)
- [Nginx Pitfalls](http://wiki.nginx.org/Pitfalls)
