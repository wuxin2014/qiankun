由于主应用和子应用部署在不同的服务器上,静态资源的请求以及接口请求,我们使用nginx进行转发,通过主应用访问的时候,主应用的nginx将api转发到服务器a上(后端接口部署的位置),将子应用资源转发到b服务器上,再通过b服务器找到指定资源.

### 主应用nginx的配置(参考)
```
# 主应用和子应用部署不在一个服务器上
# 静态资源通过nginx转发nginx
http {
    server {
        listen 6666;
        server_name localhost;

        # 转发子应用一的静态资源到子应用一部署的服务器上
        location /child_app1/ {
            proxy_pass http://172.xx.xxx.xx:8071/child_app1/;
            proxy_set_header Host $host:$server_port;
        }
        # 转发子应用二的静态资源到子应用二部署的服务器上
        location /child_app2/ {
            proxy_pass http://172.xx.xxx.xx:8071/child_app2/;
            proxy_set_header Host $host:$server_port;
        }

        # 后端接口指向
        location /api{
            add_header 'Access-Control-Allow-Origin' $http_origin;
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, HEAD, DEL, PUT, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'web-token,app-token,Authorization,Accept,Origin,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
            add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Rnage';
            # 由于跨域请求，浏览器会先发送一个OPTIONS的预检请求，我们可以缓存第一次的预检请求的失效时间
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Max-Age' 2592000;
                add_header 'Content-Type' 'text/plain; charset=utf-8';
                add_header 'Content-Length' 0;
                return 204;
            }
            # 后端接口
            proxy_pass http://172.xx.xxx.xx:8080;
        }

        location /{
            add_header 'Access-Control-Allow-Origin' $http_origin;
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, HEAD, DEL, PUT, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'web-token,app-token,Authorization,Accept,Origin,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
            add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Rnage';
            # 由于跨域请求，浏览器会先发送一个OPTIONS的预检请求，我们可以缓存第一次的预检请求的失效时间
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Max-Age' 2592000;
                add_header 'Content-Type' 'text/plain; charset=utf-8';
                add_header 'Content-Length' 0;
                return 204;
            }
            # 主应用 前端打包文件的路径
            alias /user/html/dist/;
            index index.html index.htm;
            # 刷新页面差找页面,防止404
            try_files $uri $uri/ /index.html;
        }
    }
}
```


### 子应用nginx的配置(参考)
```
# 后端接口指向
location /api{
    add_header 'Access-Control-Allow-Origin' $http_origin;
    add_header 'Access-Control-Allow-Credentials' 'true';
    add_header 'Access-Control-Allow-Methods' 'GET, POST, HEAD, DEL, PUT, OPTIONS';
    add_header 'Access-Control-Allow-Headers' 'web-token,app-token,Authorization,Accept,Origin,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
    add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Rnage';
    # 由于跨域请求，浏览器会先发送一个OPTIONS的预检请求，我们可以缓存第一次的预检请求的失效时间
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Max-Age' 2592000;
        add_header 'Content-Type' 'text/plain; charset=utf-8';
        add_header 'Content-Length' 0;
        return 204;
    }
    # 后端接口
    # proxy_pass ENV_APP1_GATEWAY;//自定义环境变量
    proxy_pass http://172.xx.xxx.xx:8080;
}

# 前端静态资源
location /child_app1{
    add_header 'Access-Control-Allow-Origin' $http_origin;
    add_header 'Access-Control-Allow-Credentials' 'true';
    add_header 'Access-Control-Allow-Methods' 'GET, POST, HEAD, DEL, PUT, OPTIONS';
    add_header 'Access-Control-Allow-Headers' 'web-token,app-token,Authorization,Accept,Origin,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
    add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Rnage';
    # 由于跨域请求，浏览器会先发送一个OPTIONS的预检请求，我们可以缓存第一次的预检请求的失效时间
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Max-Age' 2592000;
        add_header 'Content-Type' 'text/plain; charset=utf-8';
        add_header 'Content-Length' 0;
        return 204;
    }
    # 子应用 前端打包文件的路径
    alias /user/html/dist/;
    index index.html index.htm;
    # 刷新页面差找页面,防止404
    try_files $uri $uri/ /child_app1/index.html;
}
```


#### https://vue-js.com/topic/60de76a84590fe0031e59ecb