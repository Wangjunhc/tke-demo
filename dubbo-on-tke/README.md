## Table of Contents

- [1. QCBM 项目介绍](#user-content-1-qcbm-项目介绍)
- [2. 项目资源说明](#user-content-2-项目资源说明)
- [3. TKE 上部署 QCBM](#user-content-3-tke-上部署-qcbm)
  * [3.1 创建 k8s 集群 qcbm](#user-content-32-创建-k8s-集群-qcbm)
  * [3.2 创建基础配置](#user-content-33-创建基础配置)
  * [3.3 部署 qcbm 服务](#user-content-34-部署-qcbm-服务)
- [4. 创建 CLS 日志服务](#user-content-4-创建-cls-日志服务)
  * [4.1 开启容器日志采集功能](#user-content-41-开启容器日志采集功能)
  * [4.2 创建日志集](#user-content-42-创建日志集)
  * [4.3 创建日志主题](#user-content-43-创建日志主题)
  * [4.4 创建容器日志采集规则](#user-content-44-创建容器日志采集规则)
  * [4.5 创建日志索引并查看日志](#user-content-45-创建日志索引并查看日志)


## 1. QCBM 项目介绍

Q云书城（QCBM）是一个基于腾讯云原生产品之上，使用 dubbo-2.7.8 框架开发的一个网上书城 Demo 项目。

![](pic/qcbm.gif)

Q云书城 项目由以下部分组成：

- **QCBM-Front** ：使用 react 开发的前端项目，基于 nginx 官方提供的 [1.19.8 docker 镜像](https://hub.docker.com/_/nginx) 构建和部署。
- **QCBM-Gateway** ：API 网关，接受前端的 http 请求，并转化为后台的 dubbo 请求。
- **User-Service** ：基于 dubbo 的微服务，提供用户注册、登录、鉴权等功能。
- **Favorites-Service** ：基于 dubbo 的微服务，提供用户图书收藏功能。
- **Order-Service** ：基于 dubbo 的微服务，提供用户订单生成和查询等功能。
- **Store-Service** ：基于 dubbo 的微服务，提供图书信息的存储等功能。

Q云书城 的所有项目均做了容器化，最终运行于腾讯云的 [容器服务 TKE ](https://cloud.tencent.com/product/tke) 上。本项目所采用的腾讯云产品及部署架构如下：

- **网路架构**：整个项目采用一个 VPC，并划分为两个子网：Subnet-K8S 中部署 QCBM 的应用服务，所有服务都进行了容器化，运行在 TKE 上；Subnet-Basic 中部署了有状态的基础服务，包括 Dubbo 的服务注册中心 Nacos，MySQL 和 Redis 等。
- **CDB**：选用的是 MySQL 5.7，双节点，主从直接异步复制数据。
- **Redis**：选用的 5.0 标准版，一主一备模式。
- **CVM**：选用的 SA2 标准版，运行CentOS 8.0，开启 置放群组功能。
- **TKE**：选择的是 1.18.4 版的 kubernetes，并使用 Global Router；Kube-proxy 选择的是 iptables，运行时版本为 18.6。K8S Master 采用平台托管，Worker 启用置放群组功能。
- **TSE**：微服务引擎：用于构建nacos集群。
- **TCOP**：可观测平台：用于监控微服务调用
- **CLS**：微服务日志收集

![](pic/qcbm-arch.png)

以上是本文中 QCBM 项目使用的环境资源，运行 QCBM 项目时，可选择不同的配置方式。运行 QCBM 项目前，请先熟悉和了解 Kubernetes 原理和 k8s 集群管理的命令行工具 [kubectl](https://kubernetes.io/zh/docs/reference/kubectl/overview/) 的基本使用。


## 2. 项目资源说明

本项目下有两个目录分别存放不同的部署 yaml

- **qcbm-ddl.sql**：MySQL 初始化脚本
- **config.yaml**: 配置 yaml
- **qcbm-dubbo.yaml**: 工作负载和服务的部署 yaml
- **log-rule.yaml**：容器日志采集规则 yaml

~~~sh
➜  qcbm-k8s git:(master) ✗ tree 
.
├── README.md
├── config.yaml
├── log-rule.yaml
├── pic
│   ...
├── qcbm-ddl.sql
└── qcbm-dubbo.yaml
~~~

## 3. TKE 上部署 QCBM


### 3.1 创建 k8s 集群 qcbm

实际部署前，需要新建个 K8S 集群。有关集群的创建，官方文档 [购买容器集群](https://cloud.tencent.com/document/product/457/9082) 中有详细说明。但有一点需要注意：在创建集群第二步 “选择机型” 时，建议开启 “置放群组功能” 可将 CVM 打散到到不同母机上，增加系统可靠性。

![](pic/zhifangqunzu.png)

创建完成后，在容器服务控制台的 [集群管理](https://console.cloud.tencent.com/tke2/cluster) 页面可以看到新建的集群信息。这里我们新建的集群名称为 qcbm-k8s-demo。

![](pic/qcbm-k8s-demo.png)

选择集群 qcbm-k8s-demo 点击进入 **基本信息** 页面，可以看到整个集群的配置信息。

![](pic/k8s-cluster-basic-info.png)

接下来为了后续可以使用kubectl工具，还需要以下两步操作：

- 开启外网访问
- 通过Kubectl连接Kubernetes集群：具体操作过程通过 容器服务 -> 集群 -> 基本信息 下拉可找到操作步骤，如下图所示

![](pic/apiserver.png)

### 3.2 创建基础配置

实际项目上 K8S 前都有很多基本配置，对于 QCBM 项目也一样，需要提前完成以下基础配置，才能成功部署：

- 创建节点池，推荐原生节点池

- 在 [Mysql 控制台](https://console.cloud.tencent.com/cdb) 创建好实例, 并使用 `qcbm-ddl.sql` 初始化；

- 在 [Redis 控制](https://console.cloud.tencent.com/redis) 创建好实例并初始化；

- 部署 Nacos 集群，在 [微服务引擎-注册配置中心-nacos](https://console.cloud.tencent.com/tse/nacos) 新建3节点nacos集群，选择子网为     subnet-basic，在实例信息页面可以查看 集群拓扑结构、内网ip、公网地址等，可直接通过控制台登陆nacos。如果想自己创建三台服务器手动搭建nacos集群，
请参考 [腾讯云旧版方案](https://github.com/TencentCloud/container-demo/tree/main/dubbo-on-tke)

![](pic/nacos1.png)
![](pic/nacos2.png)

- 在 [CLB 控制台](https://console.cloud.tencent.com/clb) 为子网 Subnet-K8S 新建个内网型的 CLB，并用该 CLB 实例 ID 替换 `qcbm-dubbo.yaml` 中的 ***lb-xxx*** (注意：有两处需要替换)

  ~~~yaml
  # 将此处的 lb-xxx 修改为子网 subnet-k8s 购买的 CLB 实例 ID
  annotations:
    service.kubernetes.io/loadbalance-id: lb-xxx
  ~~~

- 在 TCOP -> [【应用监控】](https://console.cloud.tencent.com/monitor/apm/system)，单击【接入应用】->【Java】->【Skywalking】，
获取***接入点*** 和 ***Token*** 信息

  ![](pic/TCOP.png)

【bug提醒】
- 不要使用OpenTelemetry增强探针，因为在本项目service的启动文件(比如order-service.sh)中已经配置了skywalking的基本信息，如果再使用OpenTelemetry的tke自动接入，监测不会启动并且不会报错

  ![](pic/order-service.png)

- 在微服务引擎中创建的nacos默认密码并不是nacos，而本项目中nacos密码定为了nacos（该部分设定在application.properties中给出）并且没有在qcbm-dubbo.yaml中给出相应的NACOS_ACCOUNT和NACOS_PASSWORD，解决办法是直接把nacos密码改为nacos，同时设置公网访问的白名单只有本机


- 修改 `config.yaml` 中下面变量的值：
  - Nacos 地址：NACOS_HOST
  - Mysql 地址：MYSQL_HOST
  - Redis 地址：REDIS_HOST
  - Mysql 账号：MYSQL_ACCOUNT （需要先转换为 Base64 编码格式）
  - Mysql 密码：MYSQL_PASSWORD （需要先转换为 Base64 编码格式）
  - Redis 密码：REDIS_PASSWORD （需要先转换为 Base64 编码格式）
  - 用 skywalking 接入点 更新 SW_AGENT_COLLECTOR_BACKEND_SERVICES 变量
  - 用 skywalking Token 更新 SW_AGENT_AUTHENTICATION （需要先转换为 Base64 编码格式）
 

### 3.3 部署 qcbm 服务

部署步骤如下：

- 创建 config

    ~~~sh
    kubectl apply -f config.yaml
    ~~~

- 在qcmbm命名空间下创建名为qcloudregistrykey的Secret，不然Pod在运行时拉取镜像会失败（可以看到qcbm-dubbo.yaml中用到了qcloudregistrykey）
具体步骤如下：
  1. 点击【命名空间】下的【qcbm】，再点击【创建镜像仓库密钥】

  ![](pic/qcloudregistrykey.png)

  2. 选择Secret类型为Dockercfg，镜像仓库域名为ccr.ccs.tencentyun.com

  ![](pic/qcloud2.png)


- 创建后端服务

  ~~~sh
  kubectl apply -f qcbm-dubbo.yaml
  ~~~

【bug提醒】
- 先导入config.yaml，再导入qcbm-dubbo.yaml，因为qcbm-dubbo.yaml中用到的ConfigMap和Secret记录在config.yaml中

至此，已完成 QCBM 在 TKE 上的部署。通过 [容器服务控制台](https://console.cloud.tencent.com/tke2)，在【集群】->【服务与路由】->【Ingress】下可以看到创建的 Ingress，通过 ingress 的 VIP 就可以访问 Q云书城 的页面了。

![](pic/ingress.png)

此时，TCOP 也集成好，在 QCBM 页面上点击操作后，在 [TCOP 控制台](https://console.cloud.tencent.com/monitor/apm/system) 可以查询到各个服务的调用信息
ß
![](pic/trace.png)


## 4. 创建 CLS 日志服务

现在 CLS 上查看 QCBM 的日志，还需要进一步操作。

### 4.1 开启容器日志采集功能

- 登录 [容器服务控制台](https://console.cloud.tencent.com/tke2)，选择左侧导航栏中的【集群运维】->【功能管理】。在“功能管理”页面上方选择地域，单击需要开启日志采集的集群右侧的【设置】

  ![](pic/open-cls-1.png)

- 在“设置功能”页面，单击日志采集【编辑】，开启日志采集后确认。如下图所示：

  ![](pic/open-cls-2.png)

### 4.2 创建日志集

在日志服务控制台（ https://console.cloud.tencent.com/cls/logset）选择日志服务区域（和服务部署区域一致）。然后点击“创建日志集” 创建日志集 qcbm-logs。

![](pic/log-set.png)

### 4.3 创建日志主题

单击“日志集名称”，进入到日志主题管理页面。单击 “新增日志主题”，开始创建日志主题。QCBM 有多个后端微服务，为每个微服务建个日志主题便于日志归类。

![](pic/log-topic.png)

### 4.4 创建容器日志采集规则

用 4.3 中创建的日志主题 ID，填写到 `log-rule.yaml` 中 ***topicId*** 下，然后使用如下命令创建日志采集规则：

~~~sh
# 创建日志采集规则
kubectl apply -f log-rule.yaml
~~~

### 4.5 创建日志索引并查看日志

在容器日志服务控制台的【检索分析】中可先为日志【新建索引】：

![](pic/create-log-index.png)

然后点击 【检索分析】按钮即可查看日志。

![](pic/cls-logs.png)
