
---
title: Nginx 入门 server块 书写之 location 规则 （二）
tag: Nginx
date: 2023-4-9 14:00:00
---

# Nginx 入门 server块 书写之 location 规则 （二）

接上文，我们学习了location 路由匹配规则和优先级，今天我们深入其内部来探寻实际处理的各个属性书写方式。

### Return 指令

**return一般用于对请求的客户端直接返回响应状态码。在该作用域内return后面的所有nginx配置都是无效的。可以使用在server、location以及if配置中。除了支持跟状态码，还可以跟字符串或者url链接。**

```bash
  1 server{
  2     listen 80;
  3     server_name www.aaa.com;
  4     return 200 "hello";
  5 }
 
 
#说明：如果要想返回字符串，必须要加上状态码，否则会报错。
 
 
  1 location ^~ /aming {
  2     default_type application/json ;
  3     return 200  '{"name":"xhy","id":"100"}';
  4 }					
#返回的字符串也支持json数据。
 
 
  1 location /test {
  2     return 200 "$host $request_uri";
  3 }					
#返回的字符串也支持变量
 
 
  1 server{
  2     listen 80;
  3     server_name www.aaa.com;
  4     return http://www.baidu.com;
  5     rewrite /(.*) /abc/$1;		#该行配置位于return后，则不会被执行。
  6 }
 
# 注意：return后面的url必须是以http://或者https://开头的。
```

```bash
#常见的重定向
location = /tutorial/learning-nginx {
     return 301 $scheme://example.com/nginx/understanding-nginx
}
```

### Rewrite指令

```
rewrite regex replacement-url [flag];
regex: 正则表达式
replacement-url: 替换的URL
flag: 用于进行一些额外的处理

```

不同flag的效果：

| flag      | 说明                                  |
| :-------- | :---------------------------------- |
| last      | 停止解析，并开始搜索与更改后的`URI`相匹配的`location`; |
| break     | 中止 rewrite，不再继续匹配                   |
| redirect  | 返回临时重定向的 HTTP 状态 302                |
| permanent | 返回永久重定向的 HTTP 状态 301                |

案例

```bash
location = /nginx-tutorial 
{ 
    rewrite ^/nginx-tutorial?$ /somePage.html last; 
}


#把`https://example.com/nginx-tutorial`重写为#`https://example.com/somePage.html`

```

#####

```bash
location = /user.php 
{ 
    rewrite ^/user.php?id=([0-9]+)$ /user/$1 last; 
}
##### 动态替换案例

#把`https://www.example.com/user.php?id=11`重写为`https://exampleshop.com/user/11`

# 其中`$1`表示`regex`中第一个括号中的值，第二个括号中的值可通过`$2`获取

```

```bash
location = /
{
    if ($http_user_agent ~* (mobile|nokia|iphone|ipad|android|samsung|htc|blackberry)) {
    rewrite ^(.*) https://m.example.com$1 redirect;
    }
}

##### 手机访问重定向网址

# 把`https://www.example.com`重写为`https://m.exampleshop.com`
```

### Proxy指令

用于转发请求，常用于反向代理，注意以下细节

*   proxy\_pass的链接无`/`
*   proxy\_pass的链接有`/`

#### **第一种：proxy\_pass的链接无`/`**

**proxy\_pass中，不带『/』，则把『匹配字符串及后缀（/api/xxx）』均带给转发地址**

```bash
# 效果为：http://xxx.xxx.com/api/xxx -> http://127.0.0.1:7000/api/xxx. 转发的时候,包含了url前缀.
location ^~ /api/ { 
    proxy_pass  http://127.0.0.1:7000; 
}

# 效果与上面一致
location ^~ /api {
    proxy_pass  http://127.0.0.1:7000; 
}
```

#### 第二种：proxy\_pass的链接有`/`

proxy\_pass中，带『/』，则把『请求地址排除匹配字符串（/api/）』后，再带给转发地址

```bash
# 效果为：http://xxx.xxx.com/api/xxx --> http://127.0.0.1:7000/xxx
location ^~ /api/ {
    proxy_pass  http://127.0.0.1:7000/; # 端口后多了斜杠『/』
}

# 注意：下面的代码会导致失败，原因为『/api/xxx排除了/api』后，会把『/xxx』带给转发地址，但转发地址中已有了斜杠，结果会多了一条斜杠，报错。
# 效果为：http://xxx.xxx.com/api/xxx --> http://127.0.0.1:7000//xxx
location ^~ /api {  # 这里的匹配字符串最后少了斜杠『/』
    proxy_pass  http://127.0.0.1:7000/;
}
```

**location的修饰符为正则匹配时，proxy\_pass的地址最后不要带斜杠**

一些简单的常用的 proxy\_pass参数

```bash
location / {
      proxy_pass http://game;
      # 用户请求的时候HOST的值是game1.test.com, 那么代理服务会像后端传递请求的还是game1.test.com
      proxy_set_header Host $http_host;
      # 将$remote_addr的值放进变量X-Real-IP中，$remote_addr的值为客户端的ip
      proxy_set_header X-Real-IP $remote_addr;
      # 客户端通过代理服务访问后端服务, 后端服务通过该变量会记录真实客户端地址
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      # nginx代理与后端服务器连接超时时间(代理连接超时)
      proxy_connect_timeout 10s;
     # nginx代理等待后端服务器的响应时间 
     proxy_read_timeout 10s;
     # 后端服务器数据回传给nginx代理超时时间
     proxy_send_timeout 10s;
     # nignx会把后端返回的内容先放到缓冲区当中，然后再返回给客户端，边收边传, 不是全部接收完再传给客户  
     proxy_buffering on;
     # 设置nginx代理保存用户头信息的缓冲区大小
     proxy_buffer_size  8k;
     # proxy_buffers 缓冲区 
     proxy_buffers 8 8k;
     # 状态标记
     proxy_next_upstream http_404  http_500  http_502  http_503  http_504  http_403  http_429;
}
```

### try\_files指令

　格式1：**try\_files** *`file`* ... *`uri`*;  格式2：**try\_files** *`file`* ... =*`code`*;

> **Checks the existence of files in the specified order and uses the first found file for request processing; the processing is performed in the current context. The path to a file is constructed from the \*`file`\*parameter according to the [root](http://nginx.org/en/docs/http/ngx_http_core_module.html#root) and [alias](http://nginx.org/en/docs/http/ngx_http_core_module.html#alias) directives. It is possible to check directory’s existence by specifying a slash at the end of a name, e.g. “`$uri/`”. If none of the files were found, an internal redirect to the *****`uri`*** specified in the last parameter is made. 

*   　　关键点1：按指定的file顺序查找存在的文件，并使用第一个找到的文件进行请求处理



*   　　关键点2：查找路径是按照给定的root或alias为根路径来查找的



*   　　关键点3：如果给出的file都没有匹配到，则重新请求最后一个参数给定的uri，就是新的location匹配



*   　　关键点4：如果是格式2，如果最后一个参数是 = 404 ，若给出的file都没有匹配到，则最后返回404的响应码

```bash
location /images/ {
    root /opt/html/;
    try_files $uri   $uri/  /images/default.gif; 
}
# 比如 请求 127.0.0.1/images/test.gif 会依次查找 1.文#件/opt/html/images/test.gif   2.文件夹 /opt/html/images/test.gif/下的index文件  3. 请求127.0.0.1/images/default.gif

```

[nginx配置选项try\_files详解 - 陈一风 - 博客园 (cnblogs.com)](https://www.cnblogs.com/jedi1995/p/10900224.html)

### Nginx内置绑定变量

[使用 Nginx 内置绑定变量 · OpenResty最佳实践 (gitbooks.io)](https://moonbingbing.gitbooks.io/openresty-best-practices/content/openresty/inline_var.html)
