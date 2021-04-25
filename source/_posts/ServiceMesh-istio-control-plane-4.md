---
title: ServiceMesh之istio控制面四
date: 2021-04-25 20:36:06
tags:
---
## ServiceEntry和WorkloadEntry的event
```go
func NewServiceDiscovery(configController model.ConfigStoreCache, store model.IstioConfigStore, xdsUpdater model.XDSUpdater) *ServiceEntryStore {
	s := &ServiceEntryStore{
	}
	if configController != nil {
		configController.RegisterEventHandler(gvk.ServiceEntry, s.serviceEntryHandler)
		configController.RegisterEventHandler(gvk.WorkloadEntry, s.workloadEntryHandler)
	}
	return s
}
```
```go
func (cl *Client) RegisterEventHandler(kind resource.GroupVersionKind, handler func(model.Config, model.Config, model.Event)) {
	h, exists := cl.kinds[kind]
	if !exists {
		return
	}
	h.handlers = append(h.handlers, handler)
}
```
```go
func (h *cacheHandler) onEvent(old interface{}, curr interface{}, event model.Event) error {
	currConfig := *TranslateObject(currItem, h.schema.Resource().GroupVersionKind(), h.client.domainSuffix)
	for _, f := range h.handlers {
		f(oldConfig, currConfig, event)
	}
	return nil
}
```
```go
	i.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			incrementEvent(kind, "add")
			cl.tryLedgerPut(obj, kind)
			cl.queue.Push(func() error {
				return h.onEvent(nil, obj, model.EventAdd)
			})
		},
		...
	})

```
```go
type queueImpl struct {
    tasks   []Task
}
func (q *queueImpl) Run(stop <-chan struct{}) {
	for {
		q.cond.L.Lock()
        ...
		var task Task
		task, q.tasks = q.tasks[0], q.tasks[1:]
		q.cond.L.Unlock()

		if err := task(); err != nil {
            ...
		}
	}
}
```

### 问题点： queue的实现有点粗糙
1. queue没有去重，如果频繁的连接k8s，list到大量的资源，而处理的慢的，容易发生oom
1. queue的执行是单线程的，处理能力相对有点弱

### 优化，可以参照kubebuilder的实现进行优化