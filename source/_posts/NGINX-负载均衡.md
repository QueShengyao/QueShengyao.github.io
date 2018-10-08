---
title: NGINX 负载均衡
date: 2018-10-08 23:53:39
tags: NGINX
---
## NGINX 负载均衡
利用 NGINX 在多个服务实例中做负载均衡是 NGINX 最常用的场景之一。在将我们现在做的产品放到公司的 AWS 上的时候，我接触到了这些，并且修改了 CI team 的部分 NGINX 配置，让它能够正确地完成反向代理的工作。

## 配置
在做负载均衡前，我们首先需要定义一个 Server 组用来表示所有存在的后台服务：
```json
http {
    upstream backend {
        server backend1.example.com weight=5;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }
}
```

然后，我们需要把流量重定向到上一步定义的 backend 上去, 我们可以通过指定 proxy_pass 的值来完成这一操作：
```json
    upstream backend {
        server backend1.example.com;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }

    server {
        location / {
            proxy_pass http://backend;
        }
    }
}
```
这里我们将所有的流量重定向到 http://backend , 这将这个 NGINX 实例上的所有流量重定向到之前定义的 backend 上去。

## 负载均衡算法
当没有指定任何信息时， NGINX 默认使用了 Round Robin(轮询)算法来重定向流量。其实 NGINX 提供了多种算法来做负载均衡，下面我们来介绍一下：

### Round Robin (轮询)
在没有指定 weight(权重) 的情况下，Round Robin 会将所有请求均匀地分发给所有后台服务实例：
```json
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
}
```
这里我们没有指定权重，所以两个后台服务会收到等量的请求。但是，当指定了权重之后，NGINX 就会将权重考虑在内：
```json
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com;
}
```
在 NGINX 中，weight 默认被设置为 1。这里我们用一开始的配置举例， backend1.example.com 的权重被设置为 5，另一个的权重没设置，所以是默认值 1。我们也没有设置轮询算法，所以这时候 NGINX 会以 5：1 的比例转发请求，即 6 个请求中， 5 个被放到了 backend1.example.com 上， 有一个被发到了 backend2.example.com 上。

### Least Connections（最少连接算法）
在这个模式下，一个请求会被 NGINX 转发到当前活跃请求数量最少的服务实例上：
```json
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}
```
我们用 least_conn 来指定最少连接优先算法, NGINX 会优先转发请求到现有连接数少的那一个服务实例上。

### IP Hash (IP 哈希)
在 IP Hash 模式下，NGINX 会根据发送请求的 IP 地址的 hash 值来决定将这个请求转发给哪个后端服务实例。被 hash 的 IP 地址要么是 IPv4 地址的前三个 16 进制数或者是整个 IPv6 地址。使用这个模式的负载均衡模式可以保证来自同一个 IP 的请求被转发到同一个服务实例上。当然，这种方法在某一个后端实例发生故障时候会导致一些节点的访问出现问题。
```json
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}
```
如果某一台服务器出现故障或者无法进行服务，我们可以给它标记上 down，这样之前被转发到这台服务器上的请求就会重新进行 hash 计算并转发到新的服务实例上:
```json
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
}
```

### Generic Hash(通用哈希)
这个模式允许管理员自定义 hash 函数的输入，比如:
```json
upstream backend {
    hash $reqeust_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
}
```
在这个例子中，我们以请求中所带的 url 为 hash 的输入。
注意到这里在 hash 那一行的最后加入了 consistent 这个关键词。这个关键词会使用一种新的 hash 算法 [ketama](https://www.last.fm/user/RJ/journal/2007/04/10/rz_libketama_-_a_consistent_hashing_algo_for_memcache_clients), 该算法会让管理员添加或删除某个服务实例的时候，只有一小部分的请求会被转发到与之前不同的服务实例上去，其他请求仍然会被转发到原有的服务实例上去。

### Random (随机算法)
顾名思义，每个请求都被随机发送到某个服务实例上去:
```json
upstream backend {
    random;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
    server backend4.example.com;
}
```
Random 模式还提供了一个参数 two，当这个参数被指定时，NGINX 会先随机地选择两个服务器(考虑 weight)，然后用以下几种方法选择其中的一个服务器:

    1. `least_conn`: 最少连接数
    2. `least_time=header(NGINX PLUS only)`: 接收到 response header 的最短平均时间
    3. `least_time=last_byte(NGINX PLUS only)`: 接收到完整请求的最短平均时间
我们可以参考下面的一个例子:
```json
upstream backend {
    random two least_time=last_byte;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
    server backend4.example.com;
}
```
当环境中有多个负载均衡服务器在向后端服务转发请求时，我们可以考虑使用 Random 模式，在只有单个负载均衡服务器时，一般不建议使用 Random 模式。

### Least Time (NGINX PLUS only)
这是一个 NGINX PLUS (NGINX 的付费版) 才有的模式，可以将请求优先转发给平均响应时间较短的服务实例，它也有三个模式:

    1. `header`: 从服务器接收到第一个字节的时间
    2. `last_byte`: 从服务器接收到完整的 response 的时间
    3. `last_byte inflight`: 从服务器接收到完整地 response 的时间（考虑不完整的请求）
例子如下:
```josn
upstream backend {
    least_time header;
    server backend1.example.com;
    server  backend2.example.com;
}
```

## 总结
NGINX 提供了多种负载均衡模式，在实际使用中，需要根据实际业务需求去做尝试，分析日志来找到最适合当前场景的复杂均衡模式。
