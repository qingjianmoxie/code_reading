# 【K8s源码品读】011：Phase 1 - kube-scheduler - 了解分配pod的大致流程

## 聚焦目标

理解一个pod的被调度的大致流程 



## 目录

1. [分析Scheduler的结构体](#Scheduler)
2. [往SchedulingQueue里](#SchedulingQueue)
3. [调度一个pod对象](#scheduleOne)
   1. [调度计算结果 - ScheduleResult](#ScheduleResult)
   2. [初步推算 - Assume](#Assume)
   3. [实际绑定 - Bind](#Bind)
4. [将绑定成功后的数据更新到etcd](#update-to-etcd)
5. [pod绑定Node的总结](#Summary)



## Scheduler

在前面，我们了解了`Pod调度算法的注册`和`Informer机制来监听kube-apiserver上的资源变化`，今天这一讲，我们就将两者串联起来，看看在kube-scheduler中，Informer监听到资源变化后，如何用调度算法将pod进行调度。

```go
// 在运行 kube-scheduler 的初期，我们创建了一个Scheduler的数据结构，回头再看看有什么和pod调度算法相关的
type Scheduler struct {
	SchedulerCache internalcache.Cache
	Algorithm core.ScheduleAlgorithm

	// 获取下一个需要调度的Pod
	NextPod func() *framework.QueuedPodInfo

	Error func(*framework.QueuedPodInfo, error)
	StopEverything <-chan struct{}

	// 等待调度的Pod队列，我们重点看看这个队列是什么
	SchedulingQueue internalqueue.SchedulingQueue

	Profiles profile.Map
	scheduledPodsHasSynced func() bool
	client clientset.Interface
}

// Scheduler的实例化函数
func New(){
  var sched *Scheduler
	switch {
  // 从 Provider 创建
	case source.Provider != nil:
		sc, err := configurator.createFromProvider(*source.Provider)
		sched = sc
  // 从文件或者ConfigMap中创建
	case source.Policy != nil:
		sc, err := configurator.createFromConfig(*policy)
		sched = sc
	default:
		return nil, fmt.Errorf("unsupported algorithm source: %v", source)
	}
}

// 两个创建方式，底层都是调用的 create 函数
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	return c.create()
}
func (c *Configurator) createFromConfig(policy schedulerapi.Policy) (*Scheduler, error){
	return c.create()
}

func (c *Configurator) create() (*Scheduler, error) {
	// 实例化 podQueue
	podQueue := internalqueue.NewSchedulingQueue(
		lessFn,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(c.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(c.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodNominator(nominator),
	)
  
	return &Scheduler{
		SchedulerCache:  c.schedulerCache,
		Algorithm:       algo,
		Profiles:        profiles,
    // NextPod 函数依赖于 podQueue
		NextPod:         internalqueue.MakeNextPodFunc(podQueue),
		Error:           MakeDefaultErrorFunc(c.client, c.informerFactory.Core().V1().Pods().Lister(), podQueue, c.schedulerCache),
		StopEverything:  c.StopEverything,
    // 调度队列被赋值为podQueue
		SchedulingQueue: podQueue,
	}, nil
}

// 再看看这个调度队列的初始化函数，从命名可以看到是一个优先队列，它的实现细节暂不细看
// 结合实际情况思考下，pod会有重要程度的区分，所以调度的顺序需要考虑优先级的
func NewSchedulingQueue(lessFn framework.LessFunc, opts ...Option) SchedulingQueue {
	return NewPriorityQueue(lessFn, opts...)
}
```



SchedulingQueue

```go
// 在上面实例化Scheduler后，有个注册事件 Handler 的函数：addAllEventHandlers(sched, informerFactory, podInformer)
func addAllEventHandlers(
	sched *Scheduler,
	informerFactory informers.SharedInformerFactory,
	podInformer coreinformers.PodInformer,
) {
	/*
	函数前后有很多注册的Handler，但是和未调度pod添加到队列相关的，只有这个
	*/
	podInformer.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
      // 定义过滤函数：必须为未调度的pod
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
					return !assignedPod(t) && responsibleForPod(t, sched.Profiles)
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return !assignedPod(pod) && responsibleForPod(pod, sched.Profiles)
					}
					utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
					return false
				default:
					utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
					return false
				}
			},
     	// 增改删三个操作对应的Handler，操作到对应的Queue
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToSchedulingQueue,
				UpdateFunc: sched.updatePodInSchedulingQueue,
				DeleteFunc: sched.deletePodFromSchedulingQueue,
			},
		},
	)
}

// 牢记我们第一阶段要分析的对象：create nginx pod，所以进入这个add的操作，对应加入到队列
func (sched *Scheduler) addPodToSchedulingQueue(obj interface{}) {
	pod := obj.(*v1.Pod)
	klog.V(3).Infof("add event for unscheduled pod %s/%s", pod.Namespace, pod.Name)
  // 加入到队列
	if err := sched.SchedulingQueue.Add(pod); err != nil {
		utilruntime.HandleError(fmt.Errorf("unable to queue %T: %v", obj, err))
	}
}

// 入队操作我们清楚了，那出队呢？我们回过头去看看上面定义的NextPod的方法实现
func MakeNextPodFunc(queue SchedulingQueue) func() *framework.QueuedPodInfo {
	return func() *framework.QueuedPodInfo {
    // 从队列中弹出
		podInfo, err := queue.Pop()
		if err == nil {
			klog.V(4).Infof("About to try and schedule pod %v/%v", podInfo.Pod.Namespace, podInfo.Pod.Name)
			return podInfo
		}
		klog.Errorf("Error while retrieving next pod from scheduling queue: %v", err)
		return nil
	}
}
```



## scheduleOne

```go
// 了解入队和出队操作后，我们看一下Scheduler运行的过程
func (sched *Scheduler) Run(ctx context.Context) {
	if !cache.WaitForCacheSync(ctx.Done(), sched.scheduledPodsHasSynced) {
		return
	}
	sched.SchedulingQueue.Run()
  // 调度一个pod对象
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()
}

// 接下来scheduleOne方法代码很长，我们一步一步来看
func (sched *Scheduler) scheduleOne(ctx context.Context) {
  // podInfo 就是从队列中获取到的pod对象
	podInfo := sched.NextPod()
	// 检查pod的有效性
	if podInfo == nil || podInfo.Pod == nil {
		return
	}
	pod := podInfo.Pod
  // 根据定义的 pod.Spec.SchedulerName 查到对应的profile
	prof, err := sched.profileForPod(pod)
	if err != nil {
		klog.Error(err)
		return
	}
  // 可以跳过调度的情况，一般pod进不来
	if sched.skipPodSchedule(prof, pod) {
		return
	}

  // 调用调度算法，获取结果
	scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, prof, state, pod)
	if err != nil {
		/*
		出现调度失败的情况：
		这个时候可能会触发抢占preempt，抢占是一套复杂的逻辑，后面我们专门会讲
		目前假设各类资源充足，能正常调度
		*/
	}
	metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
	
  // assumePod 是假设这个Pod按照前面的调度算法分配后，进行验证
	assumedPodInfo := podInfo.DeepCopy()
	assumedPod := assumedPodInfo.Pod
	// SuggestedHost 为建议的分配的Host
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		// 失败就重新分配，不考虑这种情况
	}

	// 运行相关插件的代码先跳过

	// 异步绑定pod
	go func() {
    
		// 有一系列的检查工作
    
    // 真正做绑定的动作
		err := sched.bind(bindingCycleCtx, prof, assumedPod, scheduleResult.SuggestedHost, state)
		if err != nil {
			// 错误处理，清除状态并重试
		} else {
			// 打印结果，调试时将log level调整到2以上
			if klog.V(2).Enabled() {
				klog.InfoS("Successfully bound pod to node", "pod", klog.KObj(pod), "node", scheduleResult.SuggestedHost, "evaluatedNodes", scheduleResult.EvaluatedNodes, "feasibleNodes", scheduleResult.FeasibleNodes)
			}
      // metrics中记录相关的监控指标
			metrics.PodScheduled(prof.Name, metrics.SinceInSeconds(start))
			metrics.PodSchedulingAttempts.Observe(float64(podInfo.Attempts))
      metrics.PodSchedulingDuration.WithLabelValues(getAttemptsLabel(podInfo)).Observe(metrics.SinceInSeconds(podInfo.InitialAttemptTimestamp))

			// 运行绑定后的插件
			prof.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
}
```



### ScheduleResult

```go
// 调用算法下的Schedule
func New(){
  scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, prof, state, pod)
}

func (c *Configurator) create() (*Scheduler, error) {
  algo := core.NewGenericScheduler(
		c.schedulerCache,
		c.nodeInfoSnapshot,
		extenders,
		c.informerFactory.Core().V1().PersistentVolumeClaims().Lister(),
		c.disablePreemption,
		c.percentageOfNodesToScore,
	)
  return &Scheduler{
		Algorithm:       algo,
	}, nil
}

// genericScheduler 的 Schedule 的实现
func (g *genericScheduler) Schedule(ctx context.Context, prof *profile.Profile, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	// 对 pod 进行 pvc 的信息检查
	if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
		return result, err
	}
	// 对当前的信息做一个快照
	if err := g.snapshot(); err != nil {
		return result, err
	}
	// Node 节点数量为0，表示无可用节点
	if g.nodeInfoSnapshot.NumNodes() == 0 {
		return result, ErrNoNodesAvailable
	}
  // Predict阶段：找到所有满足调度条件的节点feasibleNodes，不满足的就直接过滤
	feasibleNodes, filteredNodesStatuses, err := g.findNodesThatFitPod(ctx, prof, state, pod)
	// 没有可用节点直接报错
	if len(feasibleNodes) == 0 {
		return result, &FitError{
			Pod:                   pod,
			NumAllNodes:           g.nodeInfoSnapshot.NumNodes(),
			FilteredNodesStatuses: filteredNodesStatuses,
		}
	}
	// 只有一个节点就直接选用
	if len(feasibleNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  feasibleNodes[0].Name,
			EvaluatedNodes: 1 + len(filteredNodesStatuses),
			FeasibleNodes:  1,
		}, nil
	}
	// Priority阶段：通过打分，找到一个分数最高、也就是最优的节点
	priorityList, err := g.prioritizeNodes(ctx, prof, state, pod, feasibleNodes)
	host, err := g.selectHost(priorityList)

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(filteredNodesStatuses),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}

/*
Predict 和 Priority 是选择调度节点的两个关键性步骤， 它的底层调用了各种algorithm算法。我们暂时不细看。
以我们前面讲到过的 NodeName 算法为例，节点必须与 NodeName 匹配，它是属于Predict阶段的。
*/
```



### Assume

```go
func (sched *Scheduler) assume(assumed *v1.Pod, host string) error {
  // 将 host 填入到 pod spec字段的nodename，假定分配到对应的节点上
	assumed.Spec.NodeName = host
  // 调用 SchedulerCache 下的 AssumePod
	if err := sched.SchedulerCache.AssumePod(assumed); err != nil {
		klog.Errorf("scheduler cache AssumePod failed: %v", err)
		return err
	}
	if sched.SchedulingQueue != nil {
		sched.SchedulingQueue.DeleteNominatedPodIfExists(assumed)
	}
	return nil
}

// 回头去找 SchedulerCache 初始化的地方
func (c *Configurator) create() (*Scheduler, error) {
	return &Scheduler{
		SchedulerCache:  c.schedulerCache,
	}, nil
}

func New() (*Scheduler, error) {
  // 这里就是初始化的实例 schedulerCache
	schedulerCache := internalcache.New(30*time.Second, stopEverything)
	configurator := &Configurator{
		schedulerCache:           schedulerCache,
	}
}

// 看看AssumePod做了什么
func (cache *schedulerCache) AssumePod(pod *v1.Pod) error {
  // 获取 pod 的 uid
	key, err := framework.GetPodKey(pod)
	if err != nil {
		return err
	}
	// 加锁操作，保证并发情况下的一致性
	cache.mu.Lock()
	defer cache.mu.Unlock()
  // 根据 uid 找不到 pod 当前的状态
	if _, ok := cache.podStates[key]; ok {
		return fmt.Errorf("pod %v is in the cache, so can't be assumed", key)
	}

  // 把 Assume Pod 的信息放到对应 Node 节点中
	cache.addPod(pod)
  // 把 pod 状态设置为 Assume 成功
	ps := &podState{
		pod: pod,
	}
	cache.podStates[key] = ps
	cache.assumedPods[key] = true
	return nil
}
```



### Bind

```go
func (sched *Scheduler) bind(ctx context.Context, prof *profile.Profile, assumed *v1.Pod, targetNode string, state *framework.CycleState) (err error) {
	start := time.Now()
  // 把 assumed 的 pod 信息保存下来
	defer func() {
		sched.finishBinding(prof, assumed, targetNode, start, err)
	}()
	// 阶段1： 运行扩展绑定进行验证，如果已经绑定报错
	bound, err := sched.extendersBinding(assumed, targetNode)
	if bound {
		return err
	}
  // 阶段2：运行绑定插件验证状态
	bindStatus := prof.RunBindPlugins(ctx, state, assumed, targetNode)
	if bindStatus.IsSuccess() {
		return nil
	}
	if bindStatus.Code() == framework.Error {
		return bindStatus.AsError()
	}
	return fmt.Errorf("bind status: %s, %v", bindStatus.Code().String(), bindStatus.Message())
}
```



## Update To Etcd

```go
// 这块的代码我不做细致的逐层分析了，大家根据兴趣自行探索
func (b DefaultBinder) Bind(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) *framework.Status {
	klog.V(3).Infof("Attempting to bind %v/%v to %v", p.Namespace, p.Name, nodeName)
	binding := &v1.Binding{
		ObjectMeta: metav1.ObjectMeta{Namespace: p.Namespace, Name: p.Name, UID: p.UID},
		Target:     v1.ObjectReference{Kind: "Node", Name: nodeName},
	}
  // ClientSet就是访问kube-apiserver的客户端，将数据更新上去
	err := b.handle.ClientSet().CoreV1().Pods(binding.Namespace).Bind(ctx, binding, metav1.CreateOptions{})
	if err != nil {
		return framework.NewStatus(framework.Error, err.Error())
	}
	return nil
}

```



## Summary

今天这一次分享比较长，我们一起来总结一下：

1. Pod的调度是通过一个队列`SchedulingQueue`异步工作的
   1. 监听到对应pod事件后，放入队列
   2. 有个消费者从队列中获取pod，进行调度
2. 单个pod的调度主要分为3个步骤：
   1. 根据Predict和Priority两个阶段，调用各自的算法插件，选择最优的Node
   2. Assume这个Pod被调度到对应的Node，保存到cache
   3. 用extender和plugins进行验证，如果通过则绑定
3. 绑定成功后，将数据通过client向kube-apiserver发送，更新etcd