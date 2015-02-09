# Nginx Configuration Snippets
A collection of useful Nginx configuration snippets inspired by
[.htaccess snippets](https://github.com/phanan/htaccess).

## Table of Contents
- [Rewrite and Redirection](#rewrite-and-redirection)

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

Note that this also works for HTTPS site since we are using the `$scheme` variable.

### Force no-www
Again, the [right way](http://nginx.org/en/docs/http/converting_rewrite_rules.html)
is to define a separated server for the naked domain and redirect it.

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
You probably don't want to add that trailing slash to every URL.
(Source)[http://stackoverflow.com/questions/645853/add-slash-to-the-end-of-every-url-need-rewrite-rule-for-nginx].

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
