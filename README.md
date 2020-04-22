前言
前段时间死了好大一片服务器，眼看这么多服务器砸手里，简直血本无归。为了能活得更久，潜心研究了V2Ray，踩过了不少坑，整理一下。

程序选型
单用户自己的服务器安装个V2Ray使用很方便，官方或者233boy的一键脚本安装上就可以，但要对接到面板，一开始折腾了不少精力。

如果VPS是KVM的，那直接用人家做好的docker也行，无非就是资源多耗点。如果是openvz就没法用docker了，需要使用自己配置安装版。对接面板程序有人做了Java版和go版，Java版自己的垃圾服务器上运行一段时间就会出现连不上，需要重启。最后使用了go版，目前来看很稳定，而且还做好了服务启动，方便。

面板节点设置
面板中设置V2Ray节点，额外ID，协议等，稍后服务端和客户端配置需要匹配起来，这里需要注意一点，伪装网站是访问443或80端口的时候打开的内容，所以伪装网站和节点绑定域名就是同一个！！！最多伪装内容来源可以是本地网站或者反代到其他站。

一键安装脚本
抄了大佬们一堆shell，组装了个一键脚本自己用，只支持CentOS7和Ubuntu，有bug或不灵，自己去琢磨。

wget https://raw.githubusercontent.com/828768/Shell/master/deploy_node.sh
bash deploy_node.sh
脚本选 1 自动安装并配置V2Ray为 WS+TLS 方案，天然可以网站伪装，还可以套CDN实现死鸡鸡复活。需要配置的信息如下：

伪装域名: 填你这个节点地址绑定的域名，如：sobaigu.com
申请CA证书：y 配置Caddy在线申请，否则使用自签证书
转发路径『不要带/』: 如访问地址为 http://sobaigu.com/game ，/game 即是所谓的路径，此处填 game 就可以了
V2Ray端口: 顾名思义，V2Ray监听的端口，如 10086
V2Ray额外ID: 顾名思义，如 16
面板同步端口: 从面板同步过来的用户及配置信息，传递给V2Ray的端口，如 10087
节点ID: 面板中该节点对应的ID，如 6
数据库地址: 顾名思义，如 db.sobaigu.com 或 1.1.1.1
数据库名称: 顾名思义，如 ssrpanel
数据库用户: 顾名思义，如 ssrpanel
数据库密码: 顾名思义，如 ssrpanel
以上配置都要和面板上的配置对应起来，不知道怎么配的去看胖虎的Wiki，所有的端口请都不要占用 80，443『caddy默认用了这俩端口』 和 3306『V2Ray配置里直接就这个默认了，如果数据库端口不是这个端口的自己去改』 这几个默认端口！！！选 1 安装时会自动放个2048小游戏网站，其他代理到别站的伪装请自己去手动操作。


caddy
V2Ray推荐使用 websocket+TLS 协议，外加域名伪装，用caddy比较小巧合适，配置也比nginx简单，推荐用。关键是配置，注意看注释。
caddy配置路径：/etc/caddy/Caddyfile，

DomainName {  //这个 DomainName 就是伪装域名，也就是节点绑定的域名
  root /srv/www  //伪装内容，路径根据自己实际情况改，一键安装默认是这个位置
  tls UserName@gmail.com
  gzip
  timeouts none
  proxy /game 127.0.0.1:10086 {  //带 /game 访问时转到本地V2Ray端口处理
    websocket
    header_upstream -Origin
  }
}
上面是一键脚本的默认配置，也可以如下反向代理到其他网址，如：https://sobaigu.com

DomainName {  //建议申请证书前关闭cdn，申请成功一次就可以开了
  tls UserName@gmail.com
  gzip
  timeouts none
  proxy / https://sobaigu.com {  //这个反向代理目标就是所谓的伪装域名内容源
    without /game
  }
  proxy /game 127.0.0.1:10086 {
    websocket
    header_upstream -Origin
  }
}
以上配置，当访问 DomainName 时，打开 www 下的首页或跳转到反向代理的域名，当带路径 /game 访问时，则代理到 10086 端口，这个端口也就是V2Ray的端口。

V2Ray
V2Ray配置路径：/etc/v2ray/config.json
看上去很复杂，其实搞懂了规则还好，自己看注释吧。

{
  "log": {
    "loglevel": "debug"  //调试正常后日志输出级别建议改成error
  },
  "api": {
    "tag": "api",
    "services": [
      "HandlerService",
      "LoggerService",
      "StatsService"
    ]
  },
  "stats": {},
  "inbounds": [
    {
    "listen": "127.0.0.1",
    "port": usersync_Port,  //同步面板用户及配置的端口
    "protocol": "dokodemo-door",
    "settings": {
      "address": "127.0.0.1"
    },
    "tag": "api"
    },
    {
      "tag": "proxy",
      "port": v2ray_Port,  //V2Ray代理端口，和caddy那边对起来
      "protocol": "vmess",
      "settings": {
        "clients": [],
        "disableInsecureEncryption": true,  //禁止客户端使用不安全的加密方式
        "default": {
          "level": 0,
          "alterId": alter_Id  //额外ID要和客户端最好配置一致，实测不一致也没毛病
        }
      },
      "streamSettings": {
        "network": "ws",  //用其他协议的自己去参考官方文档改
        "wsSettings": {
          "path": "/game"  //代理路径，和caddy那边对起来，实际测试删除该配置好像也行
        }
      }
    }
  ],
  "outbounds": [{
    "protocol": "freedom"
  }],
  "routing": {
    "rules": [{
      "type": "field",
      "inboundTag": [ "api" ],
      "outboundTag": "api"
    }],    
    "strategy": "rules"
  },
  "policy": {
    "levels": {
      "0": {
        "statsUserUplink": true,
        "statsUserDownlink": true
      }
    },
    "system": {
      "statsInboundUplink": true,
      "statsInboundDownlink": true
    }
  },

  "ssrpanel": {
    "nodeId": 11,  //面板分配的节点ID
    // every N seconds
    "checkRate": 300,
    // user config
    "user": {
      // inbound tag, which inbound you would like add user to
      "inboundTag": "proxy",
      "level": 0,
      "alterId": alter_Id,  //对应上面的额外ID
      "security": "none"
    },
    // db connection
    "mysql": {
      "host": "db_Host",
      "port": db_Port,  //这个默认3306应该没人会改吧？
      "user": "db_User",
      "password": "db_Password",
      "dbname": "db_Name"
    }
  }
}
可能遇到的问题
客户端加密方式
之前出现烂苹果小火箭不能正常使用的情况，发现是服务端拒绝了不安全的加密方式，经高手点拨，原来官网有说明：

"settings": {
  "clients": [],
  "disableInsecureEncryption": true,  //禁止客户端使用不安全的加密方式
  "default": {
    "level": 0,
    "alterId": alter_Id  //额外ID要和客户端最好配置一致，实测不一致也没毛病
  }
},
之前懒人配置样本里默认设置为了 true ，表示启用禁止客户端使用不安全的加密方式，解决方法是更新胖虎面板程序添加设置加密方式，或者将配置改成 false 即可：

"disableInsecureEncryption": false,  //禁止客户端使用不安全的加密方式
懒人脚本使用的配置样本已更改为 false 感谢技术佬无私分享。

Caddy启动失败
此种情况常见就是开启了 tls 但自动申请证书失败，多半是你的IP在短时间内申请次数太多或者域名入了黑名单，具体原因自行判断，本懒人脚本已将证书默认为自签，其他解决办法可参考：Caddy自动申请CA证书失败解决办法

Caddy自动申请CA证书失败解决办法
2019-10-12 | 做网站：acme: error: 403 |  875 字 |  3 分钟
文章目录
1. 现象
2. 可能原因
3. 解决办法
3.1. 更换域名
3.2. 使用手动申请证书
3.3. 使用自签名
3.4. 关闭TLS
4. 参考文档
现象
Caddy作为一个简单轻便的网站服务端环境，本来用的好好的，突然有一天发现 443 和 80 端口不在了，网站无法访问，反向代理功能不正常……查看caddy日志有如下提示：

too many registrations for this IP
或者伴随这样的提示：

caddy.service - Caddy HTTP/2 web server
   Loaded: loaded (/usr/lib/systemd/system/caddy.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sat 2019-10-12 06:30:22 CST; 4h 10min ago
     Docs: https://caddyserver.com/docs
  Process: 758 ExecStart=/usr/local/bin/caddy -log stdout -agree=true -conf=/etc/caddy/Caddyfile -root=/var/tmp (code=exited, status=1/FAILURE)
 Main PID: 758 (code=exited, status=1/FAILURE)

Oct 12 06:30:11  caddy[758]: 2019/10/12 06:30:11 [INFO] [x.y.com] AuthURL: https://acme-v02.api.letsencrypt.org/acme/authz-v3/736450281
Oct 12 06:30:11  caddy[758]: 2019/10/12 06:30:11 [INFO] [x.y.com] acme: use tls-alpn-01 solver
Oct 12 06:30:11  caddy[758]: 2019/10/12 06:30:11 [INFO] [x.y.com] acme: Trying to solve TLS-ALPN-01
Oct 12 06:30:11  caddy[758]: 2019/10/12 06:30:11 [INFO] Unable to deactivated authorizations: https://acme-v02.api.letsencrypt.org/acme/authz-v3/736450281
Oct 12 06:30:11  caddy[758]: 2019/10/12 06:30:11 [ERROR] Renewing [x.y.com]: acme: Error -> One or more domains had a problem:
Oct 12 06:30:11  caddy[758]: [x.y.com] acme: error: 403 :: urn:ietf:params:acme:error:unauthorized :: Cannot negotiate ALPN protocol "acme-tls/1" for tls-alpn-01 challenge, url:
Oct 12 06:30:11  caddy[758]: ; trying again in 10s
Oct 12 06:30:22  systemd[1]: caddy.service: main process exited, code=exited, status=1/FAILURE
Oct 12 06:30:22  systemd[1]: Unit caddy.service entered failed state.
Oct 12 06:30:22  systemd[1]: caddy.service failed.
可能原因
一般Caddy会自动通过acme申请证书，可突然就提示没授权，也不知道具体问题在哪。因为某些网站可以正常申请，某些却不行，可能跟服务器IP有关，也有可能跟网站解析有关。

解决办法
更换域名
因为申请失败的原因各异，如果出现自动签发证书失败，考虑下域名解析是不是错了，是不是被和谐掉了，先换个域名试下是不是就成功了？如果成功，那后面的事情就不用做了，能解决谁还想多折腾不是？

使用手动申请证书
自己手动从证书发行方申请证书，然后在Caddy配置中指定证书文件和秘钥对：

tls cert key
cert：证书文件。如果证书是由CA签署的，那么这个证书文件应该是一个包：服务器证书后面跟着CA证书(根证书通常不是必须的)。
key：与证书文件匹配的服务器私钥文件。 指定您自己的证书和密钥将禁用自动HTTPS，包括更改端口和将HTTP重定向到HTTPS。如果你正在管理自己的证书，那这些需要你自己去做。
使用自签名
tls self_signed
上面的语法将使用Caddy的默认TLS设置，Caddy生成并在内存中使用一个不可信的自签名证书，该证书持续7天，所以它一般仅用于本地开发。

关闭TLS
由于申请CA证书失败，Caddy就直接退出了，如果对TLS没要求，可以关闭TLS，通过以下语法配置：

tls off
禁用网站的TLS。除非你有充分的理由，否则不要推荐。关闭TLS后，自动HTTPS会被禁用，且启用默认端口(2015)，所以需要指定端口启动Caddy：

/usr/local/bin/caddy -log stdout -agree=true -conf=/etc/caddy/Caddyfile -root=/var/tmp -port=80
参考文档
https://github.com/caddyserver/caddy/wiki/v2:-Caddyfile-examples

https://dengxiaolong.com/caddy/zh/tls.html



已经装了Nginx能不能用
有Nginx的话也可以，Caddy就不要装了，Nginx上建个vhost，然后使用Nginx反代就可以了，可参考：Nginx反向代理配置规则

Nginx反向代理配置规则
2019-06-20 | 做网站 |  161 字 |  1 分钟
有些时候一些应用需要做反向代理，用caddy相对来说简单点，但一般服务器上做站还是会用nginx，所以还是整理下nginx的反向代理配置，以备不时之需。

location /game { //此处分流路径，需与后端应用配置的路径一致
    proxy_redirect off;
    proxy_pass http://127.0.0.1:10086; //10086为后端程序监听的端口
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $http_host;

    # Show realip in access.log
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
以上配置加到网站 server 配置中，当访问带 /game 路径时，会被拦截代理到本地 10086 端口。



