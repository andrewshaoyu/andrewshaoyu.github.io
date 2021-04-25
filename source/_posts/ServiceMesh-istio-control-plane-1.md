---
title: ServiceMesh之istio控制面一
date: 2021-04-22 03:24:35
tags:
---

## istio对象

1. Gateway

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
  name: shaoyu-test
  namespace: shaoyu
spec:
  selector:
    istio: ingressgateway-online-common-02
  servers:
  - hosts:
    - www.example.com
    port:
      name: example-gw
      number: 8080
      protocol: HTTP
  - hosts:
    - api.example.com
    port:
      name: example-api-gw
      number: 8080
      protocol: HTTP
```

1. VirtualService
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: vs-www.example.com
  namespace: shaoyu
spec:
  gateways:
  - shaoyu-test
  hosts:
  - www.example.com
  http:
  - match:
    - ignoreUriCase: true
      port: 8080
      uri:
        prefix: /
    name: example-app
    route:
    - destination:
        host: example-app-cluster
        port:
          number: 8080
    timeout: 120s
```

### envoy对象

1. [config.listener.v3.listener]([https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener.proto#config-listener-v3-listener](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener.proto#config-listener-v3-listener))

```json
{
    "name": "0.0.0.0_8080",
    "active_state": {
        "version_info": "2021-04-21T10:28:45Z/16403",
        "listener": {
            "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
            "name": "0.0.0.0_8080"
        },
        "address": {
            "socket_address": {
                "address": "0.0.0.0",
                "port_value": 8080
            }
        }
    }
}
```

2. [config.route.v3.RouteConfiguration]([https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route.proto#envoy-v3-api-msg-config-route-v3-routeconfiguration](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route.proto#envoy-v3-api-msg-config-route-v3-routeconfiguration))

```json
{
    "version_info": "2021-04-21T11:31:14Z/18732",
    "route_config": {
        "@type": "type.googleapis.com/envoy.config.route.v3.RouteConfiguration",
        "name": "http.8080",
        "virtual_hosts": [
            {
                "name": "www.example.org:8080",
                "domains": [
                    "www.example.org",
                    "www.example.org:*"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/",
                            "case_sensitive": false
                        },
                        "route": {
                            "cluster": "outbound|8080||example-app-cluster",
                            "timeout": "120s",
                            "max_grpc_timeout": "120s"
                        },
                        "name": "route-example-app"
                    }
                ],
                "include_request_attempt_count": true
            }
        ]
    }
}
```

## LDS源码分析

```go
func (s *DiscoveryServer) pushLds(con *Connection, push *model.PushContext, version string) error {
	rawListeners := s.ConfigGenerator.BuildListeners(con.node, push)
	response := ldsDiscoveryResponse(rawListeners, version, push.Version, con.node.RequestedTypes.LDS)
	err := con.send(response)
	if err != nil {
		recordSendError("LDS", con.ConID, ldsSendErrPushes, err)
		return err
	}
	return nil
}
```

### BuildListeners

1. 遍历node.mergedGatewy.Servers
2. 获取到port.number，生成config.listener.v3.listener
3. 推送给node

## RDS源码分析

```go
func (s *DiscoveryServer) pushRoute(con *Connection, push *model.PushContext, version string) error {
	pushStart := time.Now()
	rawRoutes := s.ConfigGenerator.BuildHTTPRoutes(con.node, push, con.Routes())
	/*
	tip: con.Route()就是route_config.name的集合
	http.8080的virtual_hosts的集合中每个子项对应约等于gateway中server.hosts(这个是近似，在处理的过程会对泛域名进行处理)。
	https的server，每一个server对应一个route_config。
	con.Route()的数量就对应下发给envoy的route_config的数量。
	*/
	response := routeDiscoveryResponse(rawRoutes, version, push.Version, con.node.RequestedTypes.RDS)
	err := con.send(response)
	if err != nil {
		return err
	}
	return nil
}
```

```go
//遍历routeNames生成对应的routeConfig，针对每一个routeConfig patch envoyfilter
for _, routeName := range routeNames {
	rc := configgen.buildGatewayHTTPRouteConfig(node, push, routeName)
	if rc != nil {
		rc = envoyfilter.ApplyRouteConfigurationPatches(networking.EnvoyFilter_GATEWAY, node, push, rc)
	} else {
		rc = &route.RouteConfiguration{
			Name:             routeName,
			VirtualHosts:     []*route.VirtualHost{},
			ValidateClusters: proto.BoolFalse,
		}
	}
	routeConfigurations = append(routeConfigurations, rc)
}
```

```go
func (configgen *ConfigGeneratorImpl) buildGatewayHTTPRouteConfig(node *model.Proxy, push *model.PushContext,
	routeName string) *route.RouteConfiguration {
	merged := node.MergedGateway
	// 根据routeName获取到所有的server，就是gateway的中所有的server
	servers := merged.ServersByRouteName[routeName]
        // routeName的划分，就是按照port来的，所以一个routeName中所有的server都是同一个port
	port := int(servers[0].Port.Number)

	nameToServiceMap := push.ServiceByHostname
	// envoy的route_config virtual_host就是一组域名下，httppath指向对应envoy cluster的路由信息描述
	// 这个信息需要根据istio的VirtualService去生成
	vHostDedupMap := make(map[host.Name]*route.VirtualHost)

	// 同一个server可能有http和https的入口，server会存在多个routeConfig中，可能会被多次命中
	for _, server := range servers {
		// 根据server查询gateway，这个地方，同一个gateway可能会被多次命中
		gatewayName := merged.GatewayNameForServer[server]
		// 取出所有的virtualServices信息，需要遍历virtualService生成http路由
		virtualServices := push.VirtualServicesForGateway(node, gatewayName)
		// 同一个vs可能被绑定到多个gateway上，也会被多次命中
		for _, virtualService := range virtualServices {
			virtualServiceHosts := host.NewNames(virtualService.Spec.(*networking.VirtualService).Hosts)
			serverHosts := host.NamesForNamespace(server.Hosts, virtualService.Namespace)

			intersectingHosts := serverHosts.Intersection(virtualServiceHosts)
			//server的host是aaa.com,但是vs是bbb.com,计算之后发现server和vs没有关联关系，所以直接continue
			if len(intersectingHosts) == 0 {
				continue
			}
			// 生成routeConfig.VirtualHost.routes
			routes, err := istio_route.BuildHTTPRoutesForVirtualService(node, push, virtualService, nameToServiceMap, port, map[string]bool{gatewayName: true})
			// 组装routeConfig.VirtualHost
			for _, hostname := range intersectingHosts {
				if vHost, exists := vHostDedupMap[hostname]; exists {
					// before merging this virtual service's routes, make sure that the existing one is not a tls redirect host
					if vHost.RequireTls == route.VirtualHost_NONE {
						vHost.Routes = append(vHost.Routes, routes...)
					}
				} else {
					newVHost := &route.VirtualHost{
						Name:                       domainName(string(hostname), port),
						Domains:                    buildGatewayVirtualHostDomains(string(hostname), port),
						Routes:                     routes,
						IncludeRequestAttemptCount: true,
					}
					vHostDedupMap[hostname] = newVHost
				}
			}
		}
	}

	var virtualHosts []*route.VirtualHost

	virtualHosts = make([]*route.VirtualHost, 0, len(vHostDedupMap))
	for _, v := range vHostDedupMap {
		v.Routes = istio_route.CombineVHostRoutes(v.Routes)
		virtualHosts = append(virtualHosts, v)
	}
	routeCfg := &route.RouteConfiguration{
		// Retain the routeName as its used by EnvoyFilter patching logic
		Name:             routeName,
		VirtualHosts:     virtualHosts,
		ValidateClusters: proto.BoolFalse,
	}
	return routeCfg
}
```

# 总结

### istio的Gateway和VirtualService就是envoy LDS和RDS的抽象

istio的抽象更加的灵活，配置更加方便

### 使用方式和rds生成的耗时探索

route_config的规模=https域名的数量+1(http.8080)

以http.8080这个routeConfig为例，为了减少遍历的次数，node上的server越少越快，每个gateway对应的virtualservice越少越快

每组域名生成一个Gateway对象，每一个gateway对象管理一个VirtualService。

在这样场景之下，生成http.8080的routeConfig,rds生成耗时不会因为多次循环放大。
