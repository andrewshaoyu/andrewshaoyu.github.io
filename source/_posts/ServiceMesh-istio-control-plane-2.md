---
title: ServiceMesh之istio控制面二 date: 2021-04-25 15:23:35 tags:
---

## istio envoyfilter对象

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
spec:
  configPatches:
    - applyTo: HTTP_FILTER // HTTP_FILTER,NETWORK_FILTER
      match: //
        context: Gateway // oneof(Gateway,SidecarInBound,SIDECAR_OUTBOUND)
        proxyMatch: // match istio version
          proxyVersion: v1.7.5
        routeConfiguration: // oneof (routeConfiguration,listener,cluster)
          portNumber: 8080
          portName: HTTP
          gateway: shaoyu-test
          vhost:
            name: www.example.com:8080
            route:
              name: example-app
              action: oneof(INSERT_BEFORE,ROUTE,REDIRECT,DIRECT_RESPONSE)
          name:
      patch:
        operation: INSERT_BEFORE
        value:
```

1. configPatch的类型

## rds envoyfilter patch

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

1. envoyfilter的patch是遍历的routeName
1. 如果一个vs既有https又有http的入口，那在patch envoyfilter的时候，一个vs下面的所有的route都会被遍历

## patch是如何做的

``` go
func ApplyRouteConfigurationPatches(patchContext networking.EnvoyFilter_PatchContext,proxy *model.Proxy,push *model.PushContext,routeConfiguration *route.RouteConfiguration) (out *route.RouteConfiguration) {
	out = routeConfiguration
	// 从pushContext中，根据proxy的信息过滤所有符合条件的envoyfilter
	efw := push.EnvoyFilters(proxy)
	// only merge is applicable for route configuration.
	...
	doVirtualHostListOperation(patchContext, efw.Patches, routeConfiguration)
	return routeConfiguration
}
```

```go
func doVirtualHostListOperation(patchContext networking.EnvoyFilter_PatchContext, patches map[networking.EnvoyFilter_ApplyTo][]*model.EnvoyFilterConfigPatchWrapper, routeConfiguration *route.RouteConfiguration) {
virtualHostsRemoved := false
    // first do removes/merges
    for _, vhost := range routeConfiguration.VirtualHosts {
        doVirtualHostOperation(patchContext, patches, routeConfiguration, vhost, &virtualHostsRemoved)
    }

// envoyFilter type VIRTUAL_HOST add and remove
...
}
```

```go
func doVirtualHostOperation(patchContext networking.EnvoyFilter_PatchContext, patches map[networking.EnvoyFilter_ApplyTo][]*model.EnvoyFilterConfigPatchWrapper, routeConfiguration *route.RouteConfiguration, virtualHost *route.VirtualHost, virtualHostRemoved *bool) {
// 遍历envoyfilter，取出类型是EnvoyFilter_VIRTUAL_HOST，类型是merge
    for _, cp := range patches[networking.EnvoyFilter_VIRTUAL_HOST] {
    if commonConditionMatch(patchContext, cp) && routeConfigurationMatch(patchContext, routeConfiguration, cp) && virtualHostMatch(virtualHost, cp) {
    if cp.Operation == networking.EnvoyFilter_Patch_REMOVE {
        ...
    } else if cp.Operation == networking.EnvoyFilter_Patch_MERGE {
        proto.Merge(virtualHost, cp.Value)
            }
        }
    }
    // 针对每一条路由，执行patch
    doHTTPRouteListOperation(patchContext, patches, routeConfiguration, virtualHost)
}
```

```go
func doHTTPRouteListOperation(patchContext networking.EnvoyFilter_PatchContext,
patches map[networking.EnvoyFilter_ApplyTo][]*model.EnvoyFilterConfigPatchWrapper,
routeConfiguration *route.RouteConfiguration, virtualHost *route.VirtualHost) {
    routesRemoved := false
    // 遍历每一条路由，进行patch，patches是通过proxy和routeName以及virtualHost(optional)关联出来的
    for index := range virtualHost.Routes {
        doHTTPRouteOperation(patchContext, patches, routeConfiguration, virtualHost, index, &routesRemoved)
    }
    // now for the adds and remove
    ...
}

```
```go
func doHTTPRouteOperation(patchContext networking.EnvoyFilter_PatchContext,
	patches map[networking.EnvoyFilter_ApplyTo][]*model.EnvoyFilterConfigPatchWrapper,
	routeConfiguration *route.RouteConfiguration, virtualHost *route.VirtualHost, routeIndex int, routesRemoved *bool) {
	for _, cp := range patches[networking.EnvoyFilter_HTTP_ROUTE] {
		//将符合条件的envoyfilter patch到route上
		if commonConditionMatch(patchContext, cp) &&
			routeConfigurationMatch(patchContext, routeConfiguration, cp) &&
			virtualHostMatch(virtualHost, cp) &&
			routeMatch(virtualHost.Routes[routeIndex], cp) {

			// different virtualHosts may share same routes pointer
			virtualHost.Routes = cloneVhostRoutes(virtualHost.Routes)
			if cp.Operation == networking.EnvoyFilter_Patch_REMOVE {
				virtualHost.Routes[routeIndex] = nil
				*routesRemoved = true
				return
			} else if cp.Operation == networking.EnvoyFilter_Patch_MERGE {
				proto.Merge(virtualHost.Routes[routeIndex], cp.Value)
			}
		}
	}
}
```
# 总结
envoyfilter的灵活性很强，但是使用起来还是有点痛苦的。 

将routes的列表和patch的列表做匹配，找到每个route需要的patch，现在有的实现上，近似于两层的循环。 在route和envoyfilter量大的情况下，patch的速度会严重影响rds build的耗时。

同理针对cluster的patch，也有这样的问题

1. 根据使用场景，对envoyfilter做预处理，可以根据route可以通过近似O(1)的时间复杂度去get到需要patch的filter
1. 遍历全量的route也是相对比较耗时的，需要进步一步的优化

