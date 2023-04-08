
---
title: Nginx 入门 server块 书写之 location 规则 （一）
tag: Nginx
date: 2023-4-9 02:17:00
---



# Nginx 入门 server块 书写之 location 规则 （一）

一个高性能的支持高并发的Web服务器，代理服务器

1.  反向代理
2.  负载均衡
3.  Web服务器
4.  安全校验，接口限流

#### 处理请求

```bash
server {            						# 第一个Server区块开始，表示一个独立的虚拟主机站点
        listen       80；      					# 提供服务的端口，默认80
        server_name  localhost；       			# 提供服务的域名主机名
        location / {            				# 第一个location区块开始
            root   html；       				# 站点的根目录，相当于Nginx的安装目录
            index  index.html index.htm；      	# 默认的首页文件，多个用空格分开
        }          								# 第一个location区块结果
}
```

接受请求后会通过listen 以及 server\_name 来匹配server模块，然后根据location匹配路径资源，一个典型的应用就是一个二级域名下的子域名全部代理到同一个端口

```bash
server {            						# 第一个Server区块开始，表示一个独立的虚拟主机站点
        listen       80；      					# 提供服务的端口，默认80
        server_name  demo1.aaa.com；       			# 提供服务的域名主机名
        location / {            				# 第一个location区块开始
            root   html1；       				# 站点的根目录，相当于Nginx的安装目录
            index  index.html index.htm；      	# 默认的首页文件，多个用空格分开
        }          								# 第一个location区块结果
}
server {            						# 第一个Server区块开始，表示一个独立的虚拟主机站点
        listen       80；      					# 提供服务的端口，默认80
        server_name  demo2.aaa.com；       			# 提供服务的域名主机名
        location / {            				# 第一个location区块开始
            root   html2；       				# 站点的根目录，相当于Nginx的安装目录
            index  index.html index.htm；      	# 默认的首页文件，多个用空格分开
        }          								# 第一个location区块结果
}
```

#### location语法

##### 先说alias 和 root 的区别

```bash
location /img/ {
	alias /var/www/image/;
}
#若按照上述配置的话，则访问/img/目录里面的文件时，ningx会自动去/var/www/image/目录找文件
location /img/ {
	root /var/www/image;
}
#若按照这种配置的话，则访问/img/目录下的文件时，nginx会去/var/www/image/img/目录下找文件
```

所以使用alias 最后一定是 以 / 结尾 .

##### 修饰符匹配

```bash
server {
    server_name website.com;
    location = /abcd {
    […]
    }
}

#  `http://website.com/abcd`**匹配**
#  `http://website.com/ABCD`**可能会匹配** ，也可以不匹配，取决于操作系统的文件系统是否大小写敏感（case-sensitive）。ps: Mac 默认是大小写不敏感的
#  `http://website.com/abcd?param1&param2`**匹配**，忽略 querystring
#  `http://website.com/abcd/`**不匹配**，带有结尾的`/`
#  `http://website.com/abcde`**不匹配**
```

```bash
server {
    server_name website.com;
    location ~ ^/abcd$ {
    […]
    }
}

# ^/abcd$这个正则表达式表示字符串必须以/开始，以$结束，中间必须是abcd
#区分大小写的正则匹配
#http://website.com/abcd匹配（完全匹配）
#http://website.com/ABCD不匹配，大小写敏感
#http://website.com/abcd?param1&param2匹配
#http://website.com/abcd/不匹配，不能匹配正则表达式
#http://website.com/abcde不匹配，不能匹配正则表达式
```

```bash
server {
    server_name website.com;
    location ~* ^/abcd$ {
    […]
    }
}
#不区分大小写的正则匹配
#http://website.com/abcd匹配 (完全匹配)
#http://website.com/ABCD匹配 (大小写不敏感)
#http://website.com/abcd?param1&param2匹配
#http://website.com/abcd/ 不匹配，不能匹配正则表达式
#http://website.com/abcde 不匹配，不能匹配正则表达式
```

&#x20;以下表格   \~ 代表自己输入的英文字母

|  匹配符  |     匹配规则     | 优先级 |
| :---: | :----------: | :-: |
|   =   |     精确匹配     |  1  |
|  ^\~  |    以某个字符开头   |  2  |
|   \~  |  区分大小写的匹配正则  |  3  |
|  \~\* |  不区分大小写的匹配正则 |  4  |
|  !\~  |  区分大小写的不匹配正则 |  5  |
| !\~\* | 不区分大小写的不匹配正则 |  6  |
|   /   | 通用匹配,所有请求都匹配 |  7  |

**	前缀匹配下，返回最长匹配的 location，与 location 所在位置顺序无关**



```bash
	server {
    server_name website.com;
	#前缀匹配
    location /doc {
        return 702;
    }
    location /docu {
        return 701;
    }
}

# curl -I website.com:8080/document 依然返回 HTTP/1.1 701
```



**	正则匹配使用文件中的顺序，找到返回**

```bash
server {
	listen 8080;
	server_name website.com;

    location ~ ^/doc[a-z]+ {
        return 701;
    }

    location ~ ^/docu[a-z]+ {
        return 702;
    }
}

# curl -I website.com:8080/document 返回 HTTP/1.1 701

server {
	listen 8080;
	server_name website.com;

    location ~ ^/docu[a-z]+ {
        return 702;
    }
    
    location ~ ^/doc[a-z]+ {
        return 701;
    }
}

# curl -I website.com:8080/document 返回 HTTP/1.1 702
```

所以当有多个匹配时匹配优先级如下

**先精确匹配，没有则查找带有 `^~`的前缀匹配，没有则进行正则匹配，最后才返回前缀匹配的结果**

```bash
location = / {
# 仅仅匹配请求 /
[ configuration A ]
}
 
location / {
# 匹配所有以 / 开头的请求。但是如果有更长的同类型的表达式，则选择更长的表达式。如果有正则表达式可以匹配，则
# 优先匹配正则表达式。
[ configuration B ]
}
 
location /documents/ {
# 匹配所有以 /documents/ 开头的请求。但是如果有更长的同类型的表达式，则选择更长的表达式。
#如果有正则表达式可以匹配，则优先匹配正则表达式。
[ configuration C ]
}
 
location ^~ /images/ {
# 匹配所有以 /images/ 开头的表达式，如果匹配成功，则停止匹配查找。所以，即便有符合的正则表达式location，也
# 不会被使用
[ configuration D ]
}
 
location ~* \.(gif|jpg|jpeg)$ {
# 匹配所有以 gif jpg jpeg结尾的请求。但是 以 /images/开头的请求，将使用 Configuration D
[ configuration E ]
}


###########请求
/ -> configuration A
/index.html -> configuration B
/documents/document.html -> configuration C
/images/1.gif -> configuration D
/documents/1.jpg -> configuration E

```

