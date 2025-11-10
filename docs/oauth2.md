---
date: 2020-07-23
tags: [tool]
---

# oauth2 认证说明

oauth2 是依赖第三方的认证方式， 例如玩游戏的时候弹出QQ登录，微信登录。
这种游戏运营商并不需要用户注册，而是直接从QQ或者微信那里获取用户的OPENID， 然后游戏游戏运营商存储并通过OPENID来识别用户

而且， 有资质的游戏运营商还可以通过玩家的openid来获取用户的信息， 例如用户的手机号，网名，年龄等信息。
但`有资质`这个是有QQ和微信来决定的，游戏运营商需要先去腾讯那里注册认证。腾讯愿意给游戏运营商分享多少信息是腾讯说了算。
所以有用户的数据在手， 腾讯稳稳当国内老大

以QQ登录为例，当玩家登录游戏时，游戏运营商先让用户访问QQ的auth2服务器，并带上游戏运营商的id。待QQ认证后会回调运营商注册的回调接口
一般为`oauth/callback`，并且带上用户的openid。 这样游戏运营商就知道是谁登录了。

如果游戏运营商需要更多用户资料时，例如注册， 运营商可以通过QQ的查询接口，密钥以及用户的openid 去查询。 这样就拉取到了用户的信息。
然后如果资料不全， 再让玩家补充，例如输入身份证号。这应该国家不准腾讯向别人分享的，必须要用户自己输入。。


## 使用github的oauth认证

1. 先去github -> developer 创建oauth应用， 输入自己的回调地址。 当用户被github认证后，会调用这个地址

2. 在服务端配置，利用一个开源 `oauth2_proxy`工具, 项目地址：

```
https://github.com/oauth2-proxy/oauth2-proxy
```

3. 配置nginx

### 配置`oauth2_proxy

之前遇到问题，一直授权不成功， 原因是cookie_secure 配成了true， 即使是https下。原因未知
```
auth_logging = true
auth_logging_format = "{{.Client}} - {{.Username}} [{{.Timestamp}}] [{{.Status}}] {{.Message}}"
## pass HTTP Basic Auth, X-Forwarded-User and X-Forwarded-Email information to upstream
pass_basic_auth = true
# pass_user_headers = true
## pass the request Host Header to upstream
## when disabled the upstream Host is used as the Host Header
pass_host_header = true
## Email Domains to allow authentication for (this authorizes any email on this domain)
## for more granular authorization use `authenticated_emails_file`
## To authorize any email addresses use "*"
# email_domains = [
#     "yourcompany.com"
# ]
email_domains="*"
## The OAuth Client ID, Secret
provider="github"
client_id = "cef54714c84e3b0c2248"
client_secret = "a96d3d94771273b5295202d03c0c2d3ca7f625dc"
## Pass OAuth Access token to upstream via "X-Forwarded-Access-Token"
pass_access_token = false
## Authenticated Email Addresses File (one email per line)
# authenticated_emails_file = ""
## Htpasswd File (optional)
## Additionally authenticate against a htpasswd file. Entries must be created with "htpasswd -s" for SHA encryption
## enabling exposes a username/login signin form
# htpasswd_file = ""
## Templates
## optional directory with custom sign_in.html and error.html
# custom_templates_dir = ""
## skip SSL checking for HTTPS requests
# ssl_insecure_skip_verify = false
## Cookie Settings
## Name     - the cookie name
## Secret   - the seed string for secure cookies; should be 16, 24, or 32 bytes
##            for use with an AES cipher when cookie_refresh or pass_access_token
##            is set
## Domain   - (optional) cookie domain to force cookies to (ie: .yourcompany.com)
## Expire   - (duration) expire timeframe for cookie
## Refresh  - (duration) refresh the cookie when duration has elapsed after cookie was initially set.
##            Should be less than cookie_expire; set to 0 to disable.
##            On refresh, OAuth token is re-validated.
##            (ie: 1h means tokens are refreshed on request 1hr+ after it was set)
## Secure   - secure cookies are only sent by the browser of a HTTPS connection (recommended)
## HttpOnly - httponly cookies are not readable by javascript (recommended)
# cookie_name = "_oauth2_proxy"
cookie_secret = "beautyfly"
cookie_domains = "beautyflying.cn"
cookie_expire = "168h"
# cookie_refresh = ""
cookie_secure = false
# cookie_httponly = true
```

可以将oauth2 配置成服务

```
[Unit]
Description = OAuth2 proxy for www blog

[Service]
Type=simple
ExecStart=/usr/bin/oauth2_proxy -config /etc/oauth2-proxy.cfg
[Install]
WantedBy=multi-user.target
```


## nginx 配置

```
    location /oauth2/ {
            proxy_pass       http://127.0.0.1:4180;
            proxy_set_header Host                    $host;
            proxy_set_header X-Real-IP               $remote_addr;
            proxy_set_header X-Scheme                $scheme;
            proxy_set_header X-Auth-Request-Redirect $request_uri;
        # or, if you are handling multiple domains:
        # proxy_set_header X-Auth-Request-Redirect $scheme://$host$request_uri;
    }

    location = /oauth2/auth {
    proxy_pass       http://127.0.0.1:4180;
    proxy_set_header Host             $host;
    proxy_set_header X-Real-IP        $remote_addr;
    proxy_set_header X-Scheme         $scheme;
    # nginx auth_request includes headers but not body
    proxy_set_header Content-Length   "";
    proxy_pass_request_body           off;
    }

    location / {

    auth_request /oauth2/auth;
    error_page 401 = /oauth2/sign_in;

    # pass information via X-User and X-Email headers to backend,
    # requires running with --set-xauthrequest flag
    auth_request_set $user   $upstream_http_x_auth_request_user;
    auth_request_set $email  $upstream_http_x_auth_request_email;
    proxy_set_header X-User  $user;
    proxy_set_header X-Email $email;

    # if you enabled --pass-access-token, this will pass the token to the backend
    auth_request_set $token  $upstream_http_x_auth_request_access_token;
    proxy_set_header X-Access-Token $token;

    # if you enabled --cookie-refresh, this is needed for it to work with auth_request
    auth_request_set $auth_cookie $upstream_http_set_cookie;
    add_header Set-Cookie $auth_cookie;

    # When using the --set-authorization-header flag, some provider's cookies can exceed the 4kb
    # limit and so the OAuth2 Proxy splits these into multiple parts.
    # Nginx normally only copies the first `Set-Cookie` header from the auth_request to the response,
    # so if your cookies are larger than 4kb, you will need to extract additional cookies manually.
    auth_request_set $auth_cookie_name_upstream_1 $upstream_cookie_auth_cookie_name_1;

    # Extract the Cookie attributes from the first Set-Cookie header and append them
    # to the second part ($upstream_cookie_* variables only contain the raw cookie content)
    if ($auth_cookie ~* "(; .*)") {
        set $auth_cookie_name_0 $auth_cookie;
        set $auth_cookie_name_1 "auth_cookie_name_1=$auth_cookie_name_upstream_1$1";
    }

    # Send both Set-Cookie headers now if there was a second part
    if ($auth_cookie_name_upstream_1) {
        add_header Set-Cookie $auth_cookie_name_0;
        add_header Set-Cookie $auth_cookie_name_1;
    }

        root   /usr/share/nginx/html/blog;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
```

## 使用baidu 的oauth 认证
