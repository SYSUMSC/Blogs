# Consul 的介绍与部署

## Consul 介绍

Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件，支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对，一致性协议采用 Raft 算法,用来保证服务的高可用，使用 GOSSIP 协议管理成员和广播消息, 并且支持 ACL 访问控制。

## 使用场景

- Docker 实例的注册与配置共享
- SaaS 应用的配置共享
- 与 Confd 服务集成，动态生成 Nginx 和 HAProxy 配置文件

## Docker 部署

使用 `docker-compose` 进行 Consul 集群部署。

```yaml
version: "3"

services:
  consul-agent-1: &consul-agent
    image: consul:latest
    networks:
      - consul-cluster
    command: "agent -retry-join consul-server-bootstrap -client 0.0.0.0"

  consul-agent-2:
    <<: *consul-agent

  consul-agent-3:
    <<: *consul-agent

  consul-server-1: &consul-server
    <<: *consul-agent
    command: "agent -server -retry-join consul-server-bootstrap -client 0.0.0.0"

  consul-server-2:
    <<: *consul-server

  consul-server-bootstrap:
    <<: *consul-agent
    ports:
      - "8400:8400"
      - "8500:8500"
      - "8600:8600"
      - "8600:8600/udp"
    command: "agent -server -bootstrap-expect 3 -ui -client 0.0.0.0"

networks: consul-cluster:
```
