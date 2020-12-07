# Multilanguage Page with URL Prefixes

<!-- TOC -->

- [Page Structure](#page-structure)
- [NGINX Configuration](#nginx-configuration)
    - [nginx.conf](#nginxconf)
    - [default.conf](#defaultconf)
- [Setup with Ansible](#setup-with-ansible)

<!-- /TOC -->

I need to configure NGINX to serve a multilanguage website with URL prefixes like `/en/`, `/fr/` and `/de/`. For this I want to use the default Gatsby Starter to create some static HTML that I can serve to test my NGINX configuration.


## Page Structure

__/public__

```bash
├── de
│  ├── index.html
│  └── sub-page
│     └── index.html
├── en
│  ├── index.html
│  └── sub-page
│     └── index.html
├── favicon.ico
├── fr
│  ├── index.html
│  └── sub-page
│     └── index.html
├── index.html
├── page-data
│  ├── app-data.json
│  ├── de
│  │  ├── page-data.json
│  │  └── sub-page
│  │     └── page-data.json
│  ├── en
│  │  ├── page-data.json
│  │  └── sub-page
│  │     └── page-data.json
│  ├── fr
│  │  ├── page-data.json
│  │  └── sub-page
│  │     └── page-data.json
│  └── index
│     └── page-data.json
```



## NGINX Configuration

__/nginx__

```bash
├── conf.d
│  ├── buffers.conf
│  ├── cache.conf
│  ├── default.conf
│  ├── gzip.conf
│  ├── header.conf
│  ├── self-signed.conf
│  ├── ssl-params.conf
│  └── timeouts.conf
├── nginx.conf
└── ssl
   ├── dhparam.pem
   ├── nginx-selfsigned.crt
   └── nginx-selfsigned.key
```



### nginx.conf

```conf
user  nginx;
worker_processes  auto;
worker_rlimit_nofile  15000;
pid  /var/run/nginx.pid;
include /usr/share/nginx/modules/*.conf;


events {
    worker_connections  2048;
    multi_accept on;
    use epoll;
}


http {
    default_type   application/octet-stream;
    # access_log   /var/log/nginx/access.log;
    # activate the server access log only when needed
    access_log     off;
    error_log      /var/log/nginx/error.log;
    # don't display server version on error pages
    server_tokens  off;
    server_names_hash_bucket_size 64;
    include        /etc/nginx/mime.types;
    sendfile       on;
    tcp_nopush     on;
    tcp_nodelay    on;

    charset utf-8;
    source_charset utf-8;
    charset_types text/xml text/plain text/vnd.wap.wml application/javascript application/rss+xml;
    
    include /etc/nginx/conf.d/default.conf;
    include /etc/nginx/conf.d/buffers.conf;
    include /etc/nginx/conf.d/timeouts.conf;
    include /etc/nginx/conf.d/gzip.conf;
    # Only activate caching in production
    # include /etc/nginx/conf.d/cache.conf;
}
```





### default.conf

__/etc/nginx/conf.d/default.conf__


```conf
server {
    listen 80;
    listen [::]:80;

    server_name 192.168.2.111;

    return 301 https://$server_name$request_uri;
}

server {
    listen      443 ssl;
    listen      [::]:443 ssl;
    include     conf.d/self-signed.conf;
    include     conf.d/ssl-params.conf;
    include     conf.d/header.conf;

    server_name 192.168.2.111;

    root         /opt/test-static/public;
    index        index.html;

    location = / {
        rewrite   ^/(.*)$  /zabbix/$1  permanent;
    }
    
    location /en/ {
        try_files  $uri $uri/ $uri.html =404; 
    }
    
    location /de/ {
        try_files  $uri $uri/ $uri.html =404; 
        
    }
    
    location /fr/ {
        try_files  $uri $uri/ $uri.html =404;
    }

    error_page  404              /404.html;
    error_page  500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```


## Setup with Ansible

```yml
---
- hosts: test
  tasks:
    - name: Aptitude Update
      apt: update_cache=yes

    - name: Add NGINX user
      user:
        name: nginx
        comment: NGINX User
        group: nginx
        shell: /bin/sh
        home: /home/nginx

    - name: Installing NGINX
      apt: pkg=nginx state=present update_cache=no

    - name: Stop NGINX Service
      service: name=nginx state=stopped 
        
    - name: Copy the NGINX Configuration
      copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
        mode: "{{item.mode}}"
      with_items:
        - { src: '/opt/test-static/nginx/nginx.conf',dest: '/etc/nginx/nginx.conf', mode: '0664'}
        - { src: '/opt/test-static/nginx/conf.d/',dest: '/etc/nginx/conf.d', mode: '0664'}
        - { src: '/opt/test-static/nginx/ssl/',dest: '/etc/nginx/ssl', mode: '0664'}
      notify:
        - Start NGINX    

  handlers:
    - name: Start NGINX
      service: name=nginx state=restarted
      # notify:
      #   - Verify that NGINX did not crash

    # - name: Verify that NGINX did not crash
    #   shell: service nginx status
    #   register: result
    #   until: result.stdout.find("active (running)") != -1
    #   retries: 5
    #   delay: 5
```