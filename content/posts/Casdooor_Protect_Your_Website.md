---
title: "使用Casdoor+OAuth-Proxy保护公网网页应用"
date: 2022-02-15T17:19:02+08:00
draft: false
toc: true
---

## 前言

前段时间把Homelab从矿渣更换为 HPE ProLiant MicroServer Gen10 Plus, 不得不说企业级的产品还是比矿渣用起来会舒服很多. 目前在Gen10 Plus上安装了ESXi作为虚拟化环境, 跑了一些DSM和OpenWRT, 还开了一个虚拟机专门跑Docker.

家里的网络是带公网IP的, 但是把应用直接暴露到公网会有各种各样的安全问题, 因此之前一直在找方法保护暴露到公网上的应用. 一开始是打算使用APISIX或者NGINX的AUTH方式来实现, 但总感觉有点高射炮打蚊子的诡异感, 因此一直没实施. 前几天无意间接触到[OAuth-Proxy](https://github.com/oauth2-proxy/oauth2-proxy)这个应用, 再加上自己也一直有参与一个SSO开源项目——[Casdoor](https://casdoor.org/)的开发, 于是便想着在Homelab上部署这套系统来保护暴露在公网上的**网页应用**.



## Casdoor的部署与设置

[Casdoor](https://casdoor.org/)是一款支持 OAuth 2.0、OIDC 和 SAML 的 UI 优先集中式身份验证/单点登录 (SSO) 平台, 可以使用Docker或直接使用源码运行, 安装步骤可见: [安装文档](https://casdoor.org/docs/basic/server-installation/). 

在安装完Casdoor后, 使用初试账户密码:`admin 123`即可进入管理后台.

![Casdoor登陆页面](https://vip1.loli.io/2022/02/15/jUbBh97QTnCWPAE.png)

在登陆Casdoor后我们可以添加一些供应商(下称idp), 方便我们进行登录. 如下图我已经添加了四个idp:

![Casdoor idp](https://vip2.loli.io/2022/02/15/RhTcLeyXrH2UuC3.png)

Casdoor支持的三方登录以及配置流程可以参考[官方文档](https://casdoor.org/docs/provider/overview). 在完成idp的添加后我们进入**用户**选项栏, 我们可以选择添加一个用户或者就使用当前的admin账号, 点击`编辑`按钮, 进行三方登录账号的绑定, 如下图我已完成所有账户的绑定:

![Casdoor账户绑定](https://vip1.loli.io/2022/02/15/ZyKEfMoXD1Yr8vi.png)

完成用户绑定后, 前往**应用**选项栏, 添加一个应用给OAuth-proxy进行使用, 目前我们不需要在这里设置过多东西, 只需要在**提供商**部分把我们刚刚添加的idp添加上去:

![应用IDP设置](https://vip2.loli.io/2022/02/15/USmEX9onqgu5jVx.png)

然后我们可以把页面里显示的**客户端ID**, **客户端密钥**记录下来, 后面会使用到.![记录相关信息](https://vip2.loli.io/2022/02/15/KOjM3Dd9aVXZuzH.png)

另外, 个人建议把**开启密码登录、启用注册**关闭, 只使用idp进行登录, 且不允许其他人进行登录:

![image-20220215182421065](https://vip1.loli.io/2022/02/15/3ikMcKNT9bVlnC4.png)

至此Casdoor的相关设置就基本结束.



## OAuth-Proxy部署及设置

OAuth-Proxy的[安装](https://oauth2-proxy.github.io/oauth2-proxy/docs/)有几种方法, 我这里是采用了最简单的预编译好的二进制文件. 把下载好的可执行文件放到Homelab中并设置好相关执行权限后, 创建一个配置文件, 可以根据[示例配置文件](https://github.com/oauth2-proxy/oauth2-proxy/blob/master/contrib/oauth2-proxy.cfg.example)进行修改. 但由于普通的配置文件的upstreams功能不太直观, 所以我使用的是[Alpha Configuration](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/alpha-config), 我使用的配置文件`config.yml`如下:

```yaml
injectRequestHeaders:
- name: X-Forwarded-Groups
  values:
  - claim: groups
- name: X-Forwarded-User
  values:
  - claim: user
- name: X-Forwarded-Email
  values:
  - claim: email
- name: X-Forwarded-Preferred-Username
  values:
  - claim: preferred_username 
- name: X-Forwarded-Access-Token
  values:
  - claim: access_token
metricsServer:
  BindAddress: ""
  SecureBindAddress: ""
  TLS: null
providers:
- ADFSConfig: {}
  approvalPrompt: force
  azureConfig:
    tenant: common
  bitbucketConfig: {}
  clientID: 7cdxxx96cb6 #这里写刚刚Casdoor获取的客户端ID
  clientSecret: 0e6fxxxa36e7ec #这里写刚刚Casdoor获取的客户端m
  githubConfig: {}
  gitlabConfig: {}
  googleConfig: {}
  id: oidc=c58c570b63c9bdbfe00c
  keycloakConfig: {}
  loginGovConfig: {}
  oidcConfig:
    emailClaim: name
    groupsClaim: groups
    insecureSkipNonce: true
    issuerURL: https://door.example.in #Casdoor的部署地址, 需要外网可以访问
    userIDClaim: name
  provider: oidc
server:
  BindAddress: "" #如果不用ssl, 这里直接写0.0.0.0:端口号
  SecureBindAddress: :7443 #OAuth-Proxy的ssl端口号
  TLS:
    Cert:
      fromFile: /path/to/cert/example.pem #证书文件
    Key:
      fromFile: /path/to/cert/example.key #证书私钥
upstreamConfig: #这后面的部分就是目前我要保护的一些内网应用
  upstreams:
  - flushInterval: 1s
    id: dsm
    passHostHeader: true
    path: ^/dsm/(.*)$
    rewriteTarget: /$1
    proxyWebSockets: true
    uri: http://192.168.2.234:5000
    insecureSkipTLSVerify: true
  - id: emby
    path: ^/emby/(.*)$
    rewriteTarget: /$1
    uri: http://192.168.2.235:8096
  - id: pt
    path: ^/pt/(.*)$
    rewriteTarget: /transmission/web/$1
    uri: http://192.168.2.234:9091
  - id: esxi
    path: ^/esxi/(.*)$
    rewriteTarget: /ui/$1
    uri: https://192.168.2.135
    insecureSkipTLSVerify: true
  - id : esxisdk
    path: ^/sdk/(.*)$
    rewriteTarget: /sdk/$1
    uri: https://192.168.2.135
    insecureSkipTLSVerify: true
  - id: esxiscreen
    path: ^/screen(.*)$
    rewriteTarget: /screen$1
    uri: https://192.168.2.135
    insecureSkipTLSVerify: true
  - id: esxiws
    path: ^/ticket/(.*)$
    rewriteTarget: /ticket/$1
    insecureSkipTLSVerify: true
    uri: https://192.168.2.135

```

还有一些配置是Alpha Configuration无法配置的, 因此还要一个普通的配置文件`setting.cfg`:

```ini
email_domains = [
    "*"
]
cookie_secret = "uq-aoO1wq4VVTvkshcNVmD6adSsuE="
cookie_secure = false
```

上面这个cookie_secret可以通过以下命令生成:

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1 | base64
```

完成上面的设置之后, 运行:

```bash
./oauth2-proxy  --config setting.cfg  --alpha-config config.yml
```

然后就可以打开设置的地址, 访问相关资源, 比如我访问我设置的emby, 会首先显示OAuth-Proxy的认证页面:

![image-20220215213522390](https://vip2.loli.io/2022/02/15/asIGT7PgptNkd3Z.png)

点击按钮, 就会跳转到Casdoor的认证页面:

![image-20220215213612224](https://vip2.loli.io/2022/02/15/weRHc7dVaGT8Lg2.png)

选择方便的方式进行登录, 然后就可以看到Emby页面了:

![image-20220215213759256](https://vip2.loli.io/2022/02/15/8crsTlVt2HM5bQm.png)



## 存在的问题

如果只打算反代一个网站, 基本没有什么问题. 如果像我这样希望用一个端口根据不同的路径访问, 首先要设置`path`和`rewriteTarget`, 大部分网站设置好这个就ok了, 但是像ESXi管理页面这种比较麻烦, 因为他不会根据当前uri来进行识别所有资源地址, 因此可以看到配置文件我esxi要反代很多个path, 但是emby这种支持比较好, 只设置根目录的转发, 其他也能很好转发.

还有一个问题是OpenWRT的网页始终无法通过反代打开, 原因还待研究.
