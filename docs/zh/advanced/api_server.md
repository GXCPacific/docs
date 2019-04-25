# 如何部署API服务

本文档介绍如何部署一个GXChain的API服务器。

## 1. 环境要求

- 系统: **macOS / Ubuntu 14.04 64-bit**, **4.4.0-63-generic** 以上内核
- 内存: 16GB+
- 硬盘: 100GB+
- 网络：10MB+带宽, 有公网独立IP

::: warning 依赖安装

* 安装ntp
``` bash
sudo apt-get install ntp
# macOS安装ntp:  brew install ntp
```

* 安装libstdc++-7-dev
```bash
# Ubuntu系统需要安装, macOS不需要
apt-get update
apt-get install software-properties-common
add-apt-repository ppa:ubuntu-toolchain-r/test
apt-get update
apt-get install libstdc++-7-dev
```

:::


## 2. 下载Release包

``` bash
# 执行这个shell脚本，会自动从github下载最新的主网程序，并解压至当前目录下
curl 'https://raw.githubusercontent.com/gxchain/gxb-core/dev_master/script/gxchain_install.sh' | bash
```

## 3. 启动节点程序，同步数据

``` bash
export LC_ALL=C

nohup ./programs/witness_node/witness_node --data-dir=trusted_node --rpc-endpoint="0.0.0.0:28090" --p2p-endpoint="0.0.0.0:6789" 1>nohup.out 2 >&1 &
```

根据上面的步骤:
- 指定区块信息保存在 `./trusted_node` 目录下
- 启动一个监听端口为28090的RPC服务，开启了API服务, 可以给钱包客户端提供RPC调用
- 启动一个监听端口为6789的P2P服务，可以作为seed node为网络中的其它节点提供连接和区块同步服务

::: tip 友情提示
- 同步区块大约需要 **30+小时**, 当然这和你的网络情况有一定关系
:::

## 4. 查看日志

``` bash
tail -f trusted_node/logs/witness.log
```

区块同步过程中，每隔1000个区块会打印一行日志； 同步到最新区块时，每3秒打印一行日志，区块号连续，日志看起来是这样的:

``` bash
2018-06-28T03:43:03 th_a:invoke handle_block         handle_block ] Got block: #10731531 time: 2018-06-28T03:43:03 latency: 60 ms from: miner11  irreversible: 10731513 (-18)			application.cpp:489
2018-06-28T03:43:06 th_a:invoke handle_block         handle_block ] Got block: #10731532 time: 2018-06-28T03:43:06 latency: 16 ms from: taffy  irreversible: 10731515 (-17)			application.cpp:489
2018-06-28T03:43:09 th_a:invoke handle_block         handle_block ] Got block: #10731533 time: 2018-06-28T03:43:09 latency: 49 ms from: david12  irreversible: 10731515 (-18)			application.cpp:489
2018-06-28T03:43:12 th_a:invoke handle_block         handle_block ] Got block: #10731534 time: 2018-06-28T03:43:12 latency: 42 ms from: miner6  irreversible: 10731516 (-18)			application.cpp:489
2018-06-28T03:43:15 th_a:invoke handle_block         handle_block ] Got block: #10731535 time: 2018-06-28T03:43:15 latency: 10 ms from: sakura  irreversible: 10731516 (-19)			application.cpp:489
2018-06-28T03:43:18 th_a:invoke handle_block         handle_block ] Got block: #10731536 time: 2018-06-28T03:43:18 latency: 57 ms from: miner9  irreversible: 10731517 (-19)			application.cpp:489
2018-06-28T03:43:21 th_a:invoke handle_block         handle_block ] Got block: #10731537 time: 2018-06-28T03:43:21 latency: 56 ms from: robin-green  irreversible: 10731517 (-20)			application.cpp:489
2018-06-28T03:43:24 th_a:invoke handle_block         handle_block ] Got block: #10731538 time: 2018-06-28T03:43:24 latency: 17 ms from: kairos  irreversible: 10731522 (-16)			application.cpp:489
2018-06-28T03:43:27 th_a:invoke handle_block         handle_block ] Got block: #10731539 time: 2018-06-28T03:43:27 latency: 21 ms from: dennis1  irreversible: 10731524 (-15)			application.cpp:489
2018-06-28T03:43:30 th_a:invoke handle_block         handle_block ] Got block: #10731540 time: 2018-06-28T03:43:30 latency: 17 ms from: aaron  irreversible: 10731524 (-16)			application.cpp:489
2018-06-28T03:43:33 th_a:invoke handle_block         handle_block ] Got block: #10731541 time: 2018-06-28T03:43:33 latency: 23 ms from: caitlin  irreversible: 10731526 (-15)			application.cpp:489
```

## 5. 测试API服务是否可用

假设你的公网IP地址为```x.x.x.x```， 调用节点的get_dynamic_global_properties API查看最新区块号：

```bash
curl POST --data '{ "jsonrpc": "2.0", "method": "call", "params": [0, "get_dynamic_global_properties", []], "id": 1 }' http://x.x.x.x:28090
```

### 附 nginx负载均衡及ssl证书配置
```
map $http_upgrade $connection_upgrade {
	default upgrade;
	'' close;
}

upstream gxb_witness {
	least_conn;
	#ip_hash;
	server 127.0.0.1:28090;
	server 172.19.20.83:28090;
}


server {
	listen 80;
	server_name node1.gxb.io;
	#deny 47.91.208.128;
	location ^~ /.well-known/acme-challenge/ {
	proxy_pass http://certificate.gxb.io/.well-known/acme-challenge/;
	proxy_redirect off;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

}
#	location = /.well-known/acme-challenge/ {
#		return 404;
#	}
	location /{
		return 302 https://node1.gxb.io$request_uri;
	}
}

server {
	listen 443 ssl;
	server_name_in_redirect on;
	server_name node1.gxb.io;
	#deny 47.91.208.128;

	ssl on;
	ssl_certificate /etc/nginx/ssl/fullchain.pem;
  ssl_certificate_key /etc/nginx/ssl/privkey.pem;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
	ssl_prefer_server_ciphers on;
	ssl_session_cache shared:SSL:1m;
	ssl_session_timeout 20m;
	proxy_read_timeout 900s;

	location / {

		#deny 47.91.208.128;
		#limit_req zone=one burst=500 nodelay;
		#limit_conn addr 500;
		#proxy_pass http://127.0.0.1:28090;
		proxy_pass http://gxb_witness;
		proxy_http_version 1.1;

		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection $connection_upgrade;
    add_header 'Access-Control-Allow-Origin' '*';
    add_header 'Access-Control-Allow-Credentials' 'true';
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
    add_header 'Access-Control-Allow-Headers' 'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With';
	}

}
```
