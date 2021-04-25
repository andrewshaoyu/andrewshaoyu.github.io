---
title: ServiceMesh之istio控制面三
date: 2021-04-25 19:57:49
tags:
---
## envoy Cluster和Endpoints
```json
    {
     "version_info": "2021-04-23T09:49:07Z/21",
     "cluster": {
      "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
      "name": "outbound_.8080_._.example-app",
      "type": "EDS",
      "eds_cluster_config": {
       "eds_config": {
        "ads": {},
        "resource_api_version": "V3"
       },
       "service_name": "outbound_.8080_._.example-app"
      },
      "connect_timeout": "10s",
      "circuit_breakers": {
       "thresholds": [
        {
         "max_connections": 4294967295,
         "max_pending_requests": 4294967295,
         "max_requests": 4294967295,
         "max_retries": 4294967295
        }
       ]
      }
     },
     "last_updated": "2021-04-23T09:49:07.419Z"
    }
```

```json
    {
     "endpoint_config": {
      "@type": "type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment",
      "cluster_name": "outbound_.80_._.example-app",
      "endpoints": [
       {
        "locality": {},
        "lb_endpoints": [
         {
          "endpoint": {
           "address": {
            "socket_address": {
             "address": "192.168.32.2",
             "port_value": 8080
            }
           },
           "health_check_config": {}
          },
          "health_status": "HEALTHY",
          "load_balancing_weight": 1
         }
        ]
       }
      ],
      "policy": {
       "overprovisioning_factor": 140
      }
     }
    }
```
### 详细描述
1. cluster name: `outbound_.80_._.example-app`,包含host和port
1. lb_endpoints，该cluster里面包含哪些ip，以及对应的端口

## ServiceEntry & WorkloadEntry
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: example-app
  namespace: shaoyu-test
spec:
  exportTo:
  - .
  hosts:
  - example-app
  location: MESH_EXTERNAL
  ports:
  - name: http-8080
    number: 8080
    protocol: TCP
  resolution: STATIC
  workloadSelector:
    labels:
      app.kubernetes.io/name: example-app
```
```yaml
apiVersion: networking.istio.io/v1beta1
kind: WorkloadEntry
metadata:
  name: example-app-1
  namespace: shaoyu-test
spec:
  address: 192.168.31.2
  labels:
    app.kubernetes.io/name: example-app
```

## istio做了什么
1. 根据ServiceEntry和WorkloadEntry生成对应的cluster和endpoints
1. ServiceEntry描述有哪些Hosts
1. ServiceEntry的selector选择WorkloadEntry.Spec.Labels
### 问题点：ServiceEntry和Cluster是一个多对多的关系
1. 构造Cluster和endpoints时，不可避免的需要将同一个namespace的ServiceEntry和WorkLoadEntry，进行两层循环的遍历，去生成对应的关系`maybeRefreshIndexes`
1. 当一个WorkloadEntry发生变化的时候，可能会影响多个Cluster。在内存中，以hostname做为Map的key进行存储Cluster和Endpoints的数据时，多线程并发去更新时，又存在竞争的问题
### 优化点
根据实际使用场景，将ServiceEntry和Cluster的关系设置成一对一，可以多线程并行的处理WorkLoadEntry
