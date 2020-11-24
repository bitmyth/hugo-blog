---
title: "Traefik"
date: 2020-11-23T22:39:54+08:00
summary: "traefik"
featured_image: "https://dpic.tiankong.com/1v/6r/QJ6247059246.jpg?x-oss-process=style/shows_794"
draft: true
---



# docker 方式启动 traefik，转发http流量配置
参考 https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-official-docker-image

## 下载配置文件

https://raw.githubusercontent.com/traefik/traefik/v2.3/traefik.sample.toml

```toml
################################################################
#
# Configuration sample for Traefik v2.
#
# For Traefik v1: https://github.com/traefik/traefik/blob/v1.7/traefik.sample.toml
#
################################################################

################################################################
# Global configuration
################################################################
[global]
  checkNewVersion = true
  sendAnonymousUsage = true

################################################################
# Entrypoints configuration
################################################################

# Entrypoints definition
#
# Optional
# Default:
[entryPoints]
  [entryPoints.web]
    address = ":80"

  [entryPoints.websecure]
    address = ":443"

  [entryPoints.mysql]
    address = ":3306"

################################################################
# Traefik logs configuration
################################################################

# Traefik logs
# Enabled by default and log to stdout
#
# Optional
#
[log]

  # Log level
  #
  # Optional
  # Default: "ERROR"
  #
 # level = "DEBUG"

  # Sets the filepath for the traefik log. If not specified, stdout will be used.
  # Intermediate directories are created if necessary.
  #
  # Optional
  # Default: os.Stdout
  #
  # filePath = "log/traefik.log"

  # Format is either "json" or "common".
  #
  # Optional
  # Default: "common"
  #
  # format = "json"

################################################################
# Access logs configuration
################################################################

# Enable access logs
# By default it will write to stdout and produce logs in the textual
# Common Log Format (CLF), extended with additional fields.
#
# Optional
#
# [accessLog]

  # Sets the file path for the access log. If not specified, stdout will be used.
  # Intermediate directories are created if necessary.
  #
  # Optional
  # Default: os.Stdout
  #
  # filePath = "/path/to/log/log.txt"

  # Format is either "json" or "common".
  #
  # Optional
  # Default: "common"
  #
  # format = "json"

################################################################
# API and dashboard configuration
################################################################

# Enable API and dashboard
[api]

  # Enable the API in insecure mode
  #
  # Optional
  # Default: false
  #
  #  insecure = true

  # Enabled Dashboard
  #
  # Optional
  # Default: true
  #
  # dashboard = false

################################################################
# Ping configuration
################################################################

# Enable ping
[ping]

  # Name of the related entry point
  #
  # Optional
  # Default: "traefik"
  #
  # entryPoint = "traefik"

################################################################
# Docker configuration backend
################################################################

# Enable Docker configuration backend
[providers.docker]

  # Docker server endpoint. Can be a tcp or a unix socket endpoint.
  #
  # Required
  # Default: "unix:///var/run/docker.sock"
  #
  # endpoint = "tcp://10.10.10.10:2375"

  # Default host rule.
  #
  # Optional
  # Default: "Host(`{{ normalize .Name }}`)"
  #
  # defaultRule = "Host(`{{ normalize .Name }}.docker.localhost`)"

  # Expose containers by default in traefik
  #
  # Optional
  # Default: true
  #
  # exposedByDefault = false
```



## 开启debug

打开配置项  level = "DEBUG"
可以看到更详细的日志

开启web ui 仪表盘
      # The Web UI (enabled by --api.insecure=true)

把下面这行最前面的注释去掉 

```toml
# insecure = true
```

## 配置file provider

```toml
[providers.file]
  directory = "/etc/traefik/config"
  watch = true
```

这段配置写到 traefik.toml 文件里
推荐使用 directory 指定配置文件路径而不是filename，并且只能用其中一个，不能同时都设置
filename and directory are mutually exclusive. The recommendation is to use directory.
参考 https://doc.traefik.io/traefik/providers/file/#filename

watch 选项设置为 true 可以使得 traefik 自动探测文件变化然后自动加载变更
参考 https://doc.traefik.io/traefik/providers/file/#watch

启动docker时 ，用 -v 选项把 file provider 的配置文件所在文件夹挂载到 docker 容器

```shell
docker run -d --rm --name traefik -p 8080:8080 -p 80:80 -p443:443    -v $PWD/traefik.toml:/etc/traefik/traefik.toml -v $PWD/config:/etc/traefik/config -v acme.json:/acme.json  traefik:v2.3
```

## 配置 http 路由

```toml
[http]
  [http.routers]
    [http.routers.Router-1]
      # By default, routers listen to every entry points
      rule = "Host(`example.com`)"
      service = "service-1"
```


这段配置写到 config/provider.toml 文件里

rule = "Host(`example.com`)" 这行表示一个路由规则，请求头的host值是 example.com 这个字符串 (Check if the request domain (host header value) targets one of the given domains.
)。rule的值需要用`包起来，或者用反斜杠转义 To set the value of a rule, use backticks ` or escaped double-quotes \".
Single quotes ' are not accepted as values are Golang's String Literals. 

参考 https://doc.traefik.io/traefik/routing/routers/#rule

请求最终是被一个具体的 service 服务的，所以必须在路由中指定提供服务的 service

## 配置 service

service 是最终处理请求的程序
参考 https://doc.traefik.io/traefik/routing/routers/#service

```toml
  [http.services]
    [http.services.service-1.loadBalancer]
      [[http.services.service-1.loadBalancer.servers]]
        url = "http://192.168.199.165:7777
```

这段配置也写到 traefik.toml 文件里

service需要指定一个负载均衡器 balancer 即使流量只转发到一个服务器
Each service has a load-balancer, even if there is only one server to forward traffic to

http://192.168.199.165:7777 这个是本地启动的http服务 ip地址是本地的ip地址

## entrypoint.sh

```bash
#!/bin/sh
set -e

# first arg is `-f` or `--some-option`
if [ "${1#-}" != "$1" ]; then
    set -- traefik "$@"
fi

# if our command is a valid Traefik subcommand, let's invoke it through Traefik instead
# (this allows for "docker run traefik version", etc)
if traefik "$1" --help >/dev/null 2>&1
then
    set -- traefik "$@"
else
    echo "= '$1' is not a Traefik command: assuming shell execution." 1>&2
fi

exec "$@"
```

## https

### 配置https 入口

```toml
 [entryPoints.websecure]
    address = ":443"
```

### 配置 http 重定向到 https

```toml
    [entryPoints.web.http]
      [entryPoints.web.http.redirections]
       [entryPoints.web.http.redirections.entryPoint]
           to = "websecure"
           scheme = "https"
```



### 运行容器时暴露 443 端口

```shell
docker run -d --rm --name traefik -p 443:443 -p 8080:8080 -p 80:80     -v $PWD/traefik.toml:/etc/traefik/traefik.toml -v $PWD/config:/etc/traefik/config -v $PWD/acme.json:/acme.json traefik:v2.3
```

### 配置 https 路由

```toml
  [http.routers.my-https-router]
      rule = "Host(`example.com`)"
      service = "service-2"
      [http.routers.my-https-router.tls]
        certResolver = "myresolver"
```

### 配置 web ui https

 ```toml
[http.routers.my-api]
       rule = "Host(`example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
       service = "api@internal"
       middlewares = ["auth"]
       [http.routers.my-api.tls]
         certResolver = "myresolver"
 ```

### 配置 basic auth

参考 https://doc.traefik.io/traefik/middlewares/basicauth/#general

在命令行用 htpasswd 命令生成密码

```shell
htpasswd -nb gsh wkkeck
gsh:$apr1$X8u10LaT$bNquJc3Ng.UDcZvD/MBJx.
```

把生成的密码写入配置

```toml
[http.middlewares.auth.basicAuth]
  users = [
    "gsh:$apr1$X8u10LaT$bNquJc3Ng.UDcZvD/MBJx.",
  ]
```



### 默认证书

没有配置let's encrypt 之前，traefik 会使用默认证书来服务https
TRAEFIK DEFAULT CERT

### acme 配置

 [certificatesResolvers.myresolver.acme]
    #email = "your-email@example.com"
traefik.toml 里加这段配置，email不能用这个原来示例里的地址，需要改成真实的邮件地址，不然 traefik 启动会报错  Contact emails @example.com are forbidden

time="2020-11-11T15:40:05Z" level=error msg="Unable to obtain ACME certificate for domains \"example.com\": cannot get ACME client acme: error: 400 :: POST :: https://acme-v02.api.letsencrypt.org/acme/new-acct :: urn:ietf:params:acme:error:invalidEmail :: Error creating new account :: invalid contact domain. Contact emails @example.com are forbidden, url: " rule="Host(`example.com`)" providerName=myresolver.acme routerName=my-https-router@filed
使用 http challenge
[certificatesResolvers.myresolver.acme.httpChallenge]
    # used during the challenge
    entryPoint = "web"

### 生成acme.json

touch acme.json
然后 -v $PWD/acme.json:/acme.json 挂载，在容器启动后看到 acme.json 文件的权限是
-rw-rw-r--    1 1000     1000             0 Nov 16 09:28 acme.json

stat acme.json 

  File: acme.json
  Size: 0         Blocks: 0          IO Block: 4096   regular empty file
Device: ca01h/51713dInode: 256385      Links: 1
Access: (0664/-rw-rw-r--)  Uid: ( 1000/ UNKNOWN)   Gid: ( 1000/ UNKNOWN)
Access: 2020-11-16 09:29:52.000000000
Modify: 2020-11-16 09:28:45.000000000
Change: 2020-11-16 09:28:45.000000000

日志中有报错

time="2020-11-16T09:29:01Z" level=error msg="The ACME resolver \"myresolver\" is skipped from the resolvers list because: unable to get ACME account: permissions 664 for acme.json are too open, please use 600"

所以为了生成权限正确的acme.json，先让容器自动生成，而不是挂载进去，等生成成功后，之后启动在挂载进去
也可以用命令 ``` chmod 600 acme.json``` 改变文件权限设置


