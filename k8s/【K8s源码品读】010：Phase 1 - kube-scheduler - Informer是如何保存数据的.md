# 【K8s源码品读】010：Phase 1 - kube-scheduler - Informer是如何保存数据的

## 聚焦目标

了解Informer在发现资源变化后，是怎么处理的



## 目录

5. [查看消费的过程](#Process)
2. [掌握Index数据结构](#Index)
3. [信息的分发distribute](#distribute)
4. [Informer的综合思考](#Summary)



## Process

```go
func (c *controller) processLoop() {
	for {
    // Pop出Object元素
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
			if c.config.RetryOnError {
				// 重新进队列
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}

// 去查看Pop的具体实现
func (f *FIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		// 调用process去处理item，然后返回
		item, ok := f.items[id]
		delete(f.items, id)
		err := process(item)
		return item, err
	}
}

// 然后去查一下 PopProcessFunc 的定义，在创建controller前
cfg := &Config{
		Process:           s.HandleDeltas,
	}

func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	for _, d := range obj.(Deltas) {
		switch d.Type {
    // 增、改、替换、同步
		case Sync, Replaced, Added, Updated:
			s.cacheMutationDetector.AddObject(d.Object)
      // 先去indexer查询
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
        // 如果数据已经存在，就执行Update逻辑
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}

				isSync := false
				switch {
				case d.Type == Sync:
					isSync = true
				case d.Type == Replaced:
					if accessor, err := meta.Accessor(d.Object); err == nil {
							isSync = accessor.GetResourceVersion() == oldAccessor.GetResourceVersion()
						}
					}
				}
      	// 分发Update事件
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
      	// 没查到数据，就执行Add操作
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
      	// 分发 Add 事件
				s.processor.distribute(addNotification{newObj: d.Object}, false)
			}
   	// 删除
		case Deleted:
    	// 去indexer删除
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
    	// 分发 delete 事件
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```



## Index

`Index` 的定义为资源的本地存储，保持与etcd中的资源信息一致。

```go
// 我们去看看Index是怎么创建的
type Indexers map[string]IndexFunc

func NewSharedInformer(lw ListerWatcher, exampleObject runtime.Object, defaultEventHandlerResyncPeriod time.Duration) SharedInformer {
  // 这里传入的是一个空的map
	return NewSharedIndexInformer(lw, exampleObject, defaultEventHandlerResyncPeriod, Indexers{})
}

func NewSharedIndexInformer(lw ListerWatcher, exampleObject runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
	sharedIndexInformer := &sharedIndexInformer{
    // 这里对空的进行了初始化
		indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
	}
	return sharedIndexInformer
}

// 生成一个map和func组合而成的Indexer
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
	return &cache{
		cacheStorage: NewThreadSafeStore(indexers, Indices{}),
		keyFunc:      keyFunc,
}

// ThreadSafeStore的底层是一个并发安全的map，具体实现我们暂不考虑
func NewThreadSafeStore(indexers Indexers, indices Indices) ThreadSafeStore {
	return &threadSafeMap{
		items:    map[string]interface{}{},
		indexers: indexers,
		indices:  indices,
	}
}
```



## distribute

```go
// 在上面的Process代码中，我们看到了将数据存储到Indexer后，调用了一个分发的函数
s.processor.distribute()

// 分发process的创建
func NewSharedIndexInformer() SharedIndexInformer {
	sharedIndexInformer := &sharedIndexInformer{
		processor:                       &sharedProcessor{clock: realClock},
	}
	return sharedIndexInformer
}

// sharedProcessor的结构
type sharedProcessor struct {
	listenersStarted bool
 	// 读写锁
	listenersLock    sync.RWMutex
  // 普通监听列表
	listeners        []*processorListener
  // 同步监听列表
	syncingListeners []*processorListener
	clock            clock.Clock
	wg               wait.Group
}

// 查看distribute函数
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()
	// 将object分发到 同步监听 或者 普通监听 的列表
	if sync {
		for _, listener := range p.syncingListeners {
			listener.add(obj)
		}
	} else {
		for _, listener := range p.listeners {
			listener.add(obj)
		}
	}
}

// 这个add的操作是利用了channel
func (p *processorListener) add(notification interface{}) {
	p.addCh <- notification
}
```



## Summary

1. `Informer` 依赖于 `Reflector` 模块，它有个组件为 xxxInformer，如 `podInformer` 
2. 具体资源的 `Informer` 包含了一个连接到``kube-apiserver`的`client`，通过`List`和`Watch`接口查询资源变更情况
3. 检测到资源发生变化后，通过`Controller` 将数据放入队列`DeltaFIFOQueue`里，生产阶段完成
4. 在`DeltaFIFOQueue`的另一端，有消费者在不停地处理资源变化的事件，处理逻辑主要分2步
   1. 将数据保存到本地存储Indexer，它的底层实现是一个并发安全的threadSafeMap
   2. 有些组件需要实时关注资源变化，会实时监听listen，就将事件分发到对应注册上来的listener上，自行处理