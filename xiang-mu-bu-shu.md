# 配置域名

```
127.0.0.1 dev-login.paascloud.net
127.0.0.1 dev-admin.paascloud.net
127.0.0.1 dev-api.paascloud.net
127.0.0.1 dev-mall.paascloud.net
127.0.0.1 paascloud-discovery
127.0.0.1 paascloud-eureka
127.0.0.1 paascloud-gateway
127.0.0.1 paascloud-monitor
127.0.0.1 paascloud-zipkin
127.0.0.1 paascloud-provider-uac
127.0.0.1 paascloud-provider-mdc
127.0.0.1 paascloud-provider-omc
127.0.0.1 paascloud-provider-opc


192.168.241.21 paascloud-db-mysql
192.168.241.21 paascloud-db-redis
192.168.241.21 paascloud-mq-rabbit
192.168.241.21 paascloud-mq-rocket
192.168.241.21 paascloud-provider-zk

192.168.241.101 paascloud-zk-01
192.168.241.102 paascloud-zk-02
192.168.241.103 paascloud-zk-03
```

### nginx配置

```
server {
    listen       80;
    server_name  dev-admin.paascloud.net;
    location / {
        proxy_pass http://localhost:7020;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
server {
    listen       80;
    server_name  dev-login.paascloud.net;
    location / {
        proxy_pass http://localhost:7010;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
server {
    listen       80;
    server_name  dev-mall.paascloud.net;
    location / {
        proxy_pass http://localhost:7030;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
server {
    listen       80;
    server_name  dev-api.paascloud.net;
    location ~ {
        proxy_pass   http://localhost:7979;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

## 微服务启动顺序

```
1. paascloud-eureka
2. paascloud-discovery
3. paascloud-provider-uac
4. paascloud-gateway
5. 剩下微服务无启动数序要求
```



