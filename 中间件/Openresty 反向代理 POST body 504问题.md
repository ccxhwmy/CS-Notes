## Openresty 反向代理 POST body 504 Gateway Time-out 问题

### 环境

- OS：Centos 6.2
- Openresty 1.21.4.1

### 现象

在 OpenResty 作为反向代理上游为 Nginx 并且返回 html 页面，并且设置了 error_page 504，向 OpenResty 反向代理发送 POST 请求并携带 body，则会出现 504 Gatway Timeout 问题

### 前置知识

`Nginx` 作为服务器响应 **HTML** 时，对于使用 `curl` 发起的请求，如果为 `POST` 请求并且携带 **body**，则会返回 `405`；

此处需要使用 `error_page` 捕获 405 并返回 200；

### 504 超时现象

#### 配置

```shell
	# 该 server 作为代理
	server {
        listen 80;
        location / {
            rewrite_by_lua_block {
                ngx.log(ngx.ERR, "enter in rewiret by lua block")
                # ！！！ 此处为必要条件 ！！！
                ngx.req.socket()
            }

            proxy_pass "http://127.0.0.1:1180/";
        }
    }

	# 该 server 作为上游服务器
    server {
        listen 1180;

		# ！！！ 此处为必要条件 ！！！
		# 捕获 405 error，返回 200
        error_page  405 =200 @405;
        location @405 {
            proxy_method GET;
            proxy_pass http://localhost:80;
        }

		# ！！！ 此处为必要条件 ！！！
		# 使用 html 作为响应
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
```

#### 请求

##### Success

启动 `OpenResty`，并发起 `GET` 请求：

```shell
curl "http://localhost:80" -vo /dev/null
* About to connect() to localhost port 80 (#0)
*   Trying ::1... Connection refused
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.44 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0< HTTP/1.1 200 OK
< Server: openresty/1.21.4.1
< Date: Sat, 17 Sep 2022 05:02:39 GMT
< Content-Type: text/html
< Content-Length: 1097
< Connection: keep-alive
< Last-Modified: Fri, 05 Aug 2022 08:32:52 GMT
< ETag: "62ecd5b4-449"
< Accept-Ranges: bytes
< 
{ [data not shown]
109  1097  109  1097    0     0  1070k      0 --:--:-- --:--:-- --:--:-- 1071k* Connection #0 to host localhost left intact

* Closing connection #0
You have new mail.           
```

启动 `OpenResty`，并发起 `POST` 请求：

```shell
curl "http://localhost:80" -XPOST -vo /dev/null
* About to connect() to localhost port 80 (#0)
*   Trying ::1... Connection refused
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> POST / HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.44 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0< HTTP/1.1 200 OK
< Server: openresty/1.21.4.1
< Date: Sat, 17 Sep 2022 05:03:22 GMT
< Content-Type: text/html
< Content-Length: 1097
< Connection: keep-alive
< Last-Modified: Fri, 05 Aug 2022 08:32:52 GMT
< ETag: "62ecd5b4-449"
< Accept-Ranges: bytes
< 
{ [data not shown]
109  1097  109  1097    0     0   733k      0 --:--:-- --:--:-- --:--:-- 1071k* Connection #0 to host localhost left intact

* Closing connection #0
```

##### 504 Gateway Time-out

启动 `OpenResty`，并发起 `POST` 携带 **body** 请求：

```shell
curl "http://localhost:80" -XPOST -d "a" -vo /dev/null
* About to connect() to localhost port 80 (#0)
*   Trying ::1... Connection refused
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> POST / HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.44 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> Content-Length: 1
> Content-Type: application/x-www-form-urlencoded
> 
} [data not shown]
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     1      0      0 --:--:--  0:00:59 --:--:--     0< HTTP/1.1 504 Gateway Time-out
< Server: openresty/1.21.4.1
< Date: Sat, 17 Sep 2022 05:06:08 GMT
< Content-Type: text/html
< Content-Length: 173
< Connection: keep-alive
< 
{ [data not shown]
* Rewinding stream by : 321 bytes on url / (size = 173, maxdownload = 173, bytecount = 0, nread = 494)
174   173  173   173    0     1      2      0  0:01:26  0:01:00  0:00:26    43* Connection #0 to host localhost left intact

* Closing connection #0
You have new mail.
```

### 对照组（全部 success）

#### 不执行 ngx.req.socket

##### 配置

```shell
	# 该 server 作为代理
	server {
        listen 80;
        location / {
            rewrite_by_lua_block {
                ngx.log(ngx.ERR, "enter in rewiret by lua block")
                # ！！！ 此处为必要条件 ！！！
                -- ngx.req.socket()
            }

            proxy_pass "http://127.0.0.1:1180/";
        }
    }

	# 该 server 作为上游服务器
    server {
        listen 1180;

		# ！！！ 此处为必要条件 ！！！
		# 捕获 405 error，返回 200
        error_page  405 =200 @405;
        location @405 {
            proxy_method GET;
            proxy_pass http://localhost:80;
        }

		# ！！！ 此处为必要条件 ！！！
		# 使用 html 作为响应
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
```

##### 请求

启动 `OpenResty`，并发起 `POST` 携带 **body** 请求：

```shell
curl "http://localhost:80" -XPOST -d "a" -vo /dev/null                                       
* About to connect() to localhost port 80 (#0)
*   Trying ::1... Connection refused
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> POST / HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.44 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> Content-Length: 1
> Content-Type: application/x-www-form-urlencoded
> 
} [data not shown]
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     1      0   1432 --:--:-- --:--:-- --:--:--  1432< HTTP/1.1 200 OK
< Server: openresty/1.21.4.1
< Date: Sat, 17 Sep 2022 05:08:53 GMT
< Content-Type: text/html
< Content-Length: 1097
< Connection: keep-alive
< Last-Modified: Fri, 05 Aug 2022 08:32:52 GMT
< ETag: "62ecd5b4-449"
< Accept-Ranges: bytes
< 
{ [data not shown]
109  1097  109  1097    0     1   487k    455 --:--:-- --:--:-- --:--:-- 1070k* Connection #0 to host localhost left intact

* Closing connection #0
```

#### 不执行 error_page

##### 配置

```shell
	# 该 server 作为代理
	server {
        listen 80;
        location / {
            rewrite_by_lua_block {
                ngx.log(ngx.ERR, "enter in rewiret by lua block")
                # ！！！ 此处为必要条件 ！！！
                ngx.req.socket()
            }

            proxy_pass "http://127.0.0.1:1180/";
        }
    }

	# 该 server 作为上游服务器
    server {
        listen 1180;

		# ！！！ 此处为必要条件 ！！！
		# 捕获 405 error，返回 200
        # error_page  405 =200 @405;
        # location @405 {
        #     proxy_method GET;
        #     proxy_pass http://localhost:80;
        # }

		# ！！！ 此处为必要条件 ！！！
		# 使用 html 作为响应
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
```

##### 请求

启动 `OpenResty`，并发起 `POST` 携带 **body** 请求：(405 但不会 Gateway Time-out)

```shell
curl "http://localhost:80" -XPOST -d "a" -vo /dev/null                                       
* About to connect() to localhost port 80 (#0)
*   Trying ::1... Connection refused
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> POST / HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.44 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> Content-Length: 1
> Content-Type: application/x-www-form-urlencoded
> 
} [data not shown]
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     1      0    680 --:--:-- --:--:-- --:--:--   680< HTTP/1.1 405 Not Allowed
< Server: openresty/1.21.4.1
< Date: Sat, 17 Sep 2022 05:16:17 GMT
< Content-Type: text/html
< Content-Length: 163
< Connection: keep-alive
< 
{ [data not shown]
* Rewinding stream by : 321 bytes on url / (size = 163, maxdownload = 163, bytecount = 0, nread = 484)
164   163  163   163    0     1  86152    528 --:--:-- --:--:-- --:--:--  158k* Connection #0 to host localhost left intact

* Closing connection #0
```

#### 使用 return 返回

##### 配置

```shell
	# 该 server 作为代理
	server {
        listen 80;
        location / {
            rewrite_by_lua_block {
                ngx.log(ngx.ERR, "enter in rewiret by lua block")
                # ！！！ 此处为必要条件 ！！！
                ngx.req.socket()
            }

            proxy_pass "http://127.0.0.1:1180/";
        }
    }

	# 该 server 作为上游服务器
    server {
        listen 1180;

		# ！！！ 此处为必要条件 ！！！
		# 捕获 405 error，返回 200
        # error_page  405 =200 @405;
        # location @405 {
        #     proxy_method GET;
        #     proxy_pass http://localhost:80;
        # }

		# ！！！ 此处为必要条件 ！！！
		# 使用 html 作为响应
        location / {
            return 200 "ok";
        }
    }
```

##### 请求

启动 `OpenResty`，并发起 `POST` 携带 **body** 请求：

```shell
curl "http://localhost:80" -XPOST -d "a" -vo /dev/null                                       
* About to connect() to localhost port 80 (#0)
*   Trying ::1... Connection refused
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> POST / HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.44 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> Content-Length: 1
> Content-Type: application/x-www-form-urlencoded
> 
} [data not shown]
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     1      0    905 --:--:-- --:--:-- --:--:--   905< HTTP/1.1 200 OK
< Server: openresty/1.21.4.1
< Date: Sat, 17 Sep 2022 05:18:13 GMT
< Content-Type: application/octet-stream
< Content-Length: 2
< Connection: keep-alive
< 
{ [data not shown]
* Rewinding stream by : 321 bytes on url / (size = 2, maxdownload = 2, bytecount = 0, nread = 323)
  0     2    0     2    0     1    981    490 --:--:-- --:--:-- --:--:--  1000* Connection #0 to host localhost left intact

* Closing connection #0
```