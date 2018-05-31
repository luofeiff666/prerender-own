# prerender 预渲染服务安装启动说明文档

[prerender](https://github.com/prerender/prerender)

## 安装&启动

1. 安装chrome(centos>=7)

2. 安装nodejs稳定版

3. 全局安装pm2,执行```npm i -g pm2``` 
	- 如果安装速度较慢，可以更换源[cnpm](https://npm.taobao.org/)
	- 什么是pm2? pm2是node进程管理工具，可以利用它来简化很多node应用管理的繁琐任务，如性能监控、自动重启、负载均衡等，更多详情查看[pm2](http://pm2.keymetrics.io/)

4. 克隆本仓库代码至服务器

5. 进入项目根目录，执行```npm i```安装项目依赖

6. 进入项目根目录，执行```npm run start```预渲染服务器即运行成功

## 项目运行环境的改变

打开根目录的package.json文件，找到如下字段
```
  "scripts": {
    "start": "export PORT=3000&&pm2 start server.js --name=prerender-htt",
    "test": "./node_modules/.bin/mocha"
  },
```
- 修改端口: 字段中的3000就是运行的端口，改变该值即可改变服务器的运行端口
- 修改服务运行名字: 为了方便在pm2中管理服务，最好是给应用提供一个标识，```--name=prerender-htt```中的```prerender-htt``就是这个标识，修改即可．

## nginx 参考配置
seo只针对搜索引擎爬虫即可，所以下面配置根据请求的useragent，如果是爬虫则走预渲染服务器，否则则走正常的代理即可
```
# 目标静态服务器
server {
    listen 88;
    server_name 10.10.22.127;
    location / {
        root /opt/haotaitai;
        try_files $uri $uri/ /index.html;
    }
}

# prerender行服务端渲染代理
server {
    listen       80;
    server_name  10.10.22.127;
    # 拦截所有请求，交由 @prerender 处理
    location / {
        try_files $uri @prerender;
    }
    location @prerender {
        set $prerender 0;
        if ($http_user_agent ~* "baiduspider|twitterbot|facebookexternalhit|rogerbot|linkedinbot|embedly|quora link preview|showyoubot|outbrain|pinterest|slackbot|vkShare|W3C_Validator") {
            set $prerender 1;
        }
        if ($args ~ "_escaped_fragment_") {
            set $prerender 1;
        }
        if ($http_user_agent ~ "Prerender") {
            set $prerender 0;
        }
        if ($uri ~* "\.(js|css|xml|less|png|jpg|jpeg|gif|pdf|doc|txt|ico|rss|zip|mp3|rar|exe|wmv|doc|avi|ppt|mpg|mpeg|tif|wav|mov|psd|ai|xls|mp4|m4a|swf|dat|dmg|iso|flv|m4v|torrent|ttf|woff|svg|eot)") {
            set $prerender 0;
        }
        #resolve using Google's DNS server to force DNS resolution and prevent caching of IPs
        resolver 8.8.8.8;
        if ($prerender = 1) {
            #setting prerender as a variable forces DNS resolution since nginx caches IPs and doesnt play well with load balancing
            # prerender服务器地址
            set $prerender "127.0.0.1:3000";
            rewrite .* /$scheme://$host$request_uri? break;
            proxy_pass http://$prerender;
        }
        if ($prerender = 0) {
            proxy_pass http://127.0.0.1:88;
        }
    }
}
```

## 如何查看配置是否成功

根据上面的nginx配置可知，预留了一个测试入口，在请求地址后面加上```?_escaped_fragment_=```即可
