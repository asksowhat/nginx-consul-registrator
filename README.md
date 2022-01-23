# 文件夹说明

```bash
.
├── cluster
│   ├── master
│   │   └── docker-compose.yml
│   ├── server
│   │   └── docker-compose.yml
│   └── slave
│       └── docker-compose.yml
├── nginx-consul-template
│   ├── Dockerfile
│   └── nginx.conf.ctmpl
└── single
    └── docker-compose.yml
```

- nginx-consul-template：生成nginx和consul-template集成的镜像，实现负载均衡，实时生成nginx配置文件

- single：单机部署动态负载均衡

- cluster：集群部署动态负载均衡，其中master为集群主节点部署，slave为集群从节点部署，server为web服务端部署

# 背景

出于兴趣，第一次用nginx实现了负载均衡，那时的做法是，将其他机器配置信息，手动填写一份，当每增加一台机器都需要人工介入修改配置，方式简单粗暴，但也确实能解决问题。后来我就在想，有没有一种方式，能够动态发现新的机器，并将其配置到nginx，这样就能实现“动态负载均衡”了。

# 所使用到的工具

- consul：用来做服务发现、健康监测、存储等

- registrator：做服务注册

- nginx：做负载均衡

- consul-template：根据consul存储的信息生成配置文件

# 环境

[play-with-k8s](https://labs.play-with-k8s.com/)免费的docker机器

或者 ubuntu20.04 须具备Docker version 20.10.7、docker-compose version 1.29.2环境

# 环境变量

## 自定义

变量名 | 说明
-|-
CURRENT_IP | 当前服务器的ip
CONSUL_NODE_NAME | consul节点名，在集群部署时需要
CONSUL_MASTER_IP | consul主节点ip

## consul提供的环境变量

变量名 | 说明
-|-
SERVICE_NAME | 服务名

在本项目中只用到了这个变量，只有SERVICE_NAME=web才可被nginx负载。

# 下载项目

```bash
git clone https://github.com/srcrs/nginx-consul-registrator.git
```

# 单机部署

![dynamic-nginx-load-balance](https://gallery-srcrs.vercel.app/blog/dynamic-nginx-load-balance.png)

需要查看当前服务器ip

```bash
#进入single文件夹
cd ./nginx-consul-registrator/single
#部署
CURRENT_IP=192.168.0.18 docker-compose up -d
```

将web服务调整为三台容器

```bash
docker-compose scale web=3
```

查看nginx配置

```bash
docker-compose exec elb cat /etc/nginx/conf.d/default.conf
```

验证，可通过nginx查看配置文件，或者访问ip:9080，查看请求 Response Headers的backendIP值。

# 集群部署

共有五台机器

192.168.0.18和192.168.0.17做负载均衡，192.168.0.16、192.168.0.15、192.168.0.14做web服务。

![cluster-elb](https://gallery-srcrs.vercel.app/blog/cluster-elb.png)

```bash
#进入nginx-consul-registrator/cluster/master文件夹
#主节点部署
CURRENT_IP=192.168.0.18 CONSUL_NODE_NAME=consul_server_master docker-compose up -d
#进入nginx-consul-registrator/cluster/slave文件夹
#从节点部署
CONSUL_MASTER_IP=192.168.0.18 CURRENT_IP=192.168.0.17 CONSUL_NODE_NAME=consul_server_slave docker-compose up -d
#进入nginx-consul-registrator/cluster/server文件夹
#web服务部署
CONSUL_MASTER_IP=192.168.0.18 CURRENT_IP=192.168.0.16 CONSUL_NODE_NAME=consul_client_01 docker-compose up -d

CONSUL_MASTER_IP=192.168.0.18 CURRENT_IP=192.168.0.15 CONSUL_NODE_NAME=consul_client_02 docker-compose up -d

CONSUL_MASTER_IP=192.168.0.18 CURRENT_IP=192.168.0.14 CONSUL_NODE_NAME=consul_client_03 docker-compose up -d
```

跳转到192.168.0.18机器

```bash
#进入nginx-consul-registrator/cluster/master文件夹
#查看nginx配置文件
docker-compose exec elb cat /etc/nginx/conf.d/default.conf
```

# 参考文章

[基于Docker + Consul + Nginx + Consul-template的服务负载均衡实现](https://juejin.cn/post/6844903623084736525)

[使用Docker、Registrator、Consul、Consul Template和Nginx实现高可扩展的Web框架](http://www.dockone.io/article/272)

[基于Consul+Registrator+Nginx实现容器服务自动发现的集群框架](https://blog.51cto.com/ganbing/2086851)

# 致谢

[consul](https://github.com/hashicorp/consul)

[nginx](https://github.com/nginx/nginx)

[registrator](https://github.com/gliderlabs/registrator)

[consul-template](https://github.com/hashicorp/consul-template)