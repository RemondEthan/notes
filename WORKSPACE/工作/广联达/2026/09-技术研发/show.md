```Weil
1. SR触发dump的过程，调用的/dump的接口    --- 公有云的接口也关掉，跟SR的人确认一下，能不能关掉
2. seaweedfs读取发生了heapbuffer越界,使用flinkx来解决这个问题，中间会有超时的情况，添加了autoConnect=true来解决
3. 入湖的SDK整理 -- 与CDE的沟通进展是什么样子的，性能的验证点在哪里？如果说我们要做compact的话，策略怎么定义
   
TODO：
1）公有云生产检查一下
2）需要测试的部分还没定
```
```Yangxu
1. 数据组件服务合并
2. 服务升级后，治理服务把kv参数中的'去掉了，导致解析是认为是数字。结果多种任务起不来。需要数据治理去修改
3. kyuubi无list权限/pod不退出
4. 服务整合以后，起任务的时用哪个sa需要好好整理  *******
```
```Licc
1. 修改alter log，在处理taskId的时候，会有多线程的安全问题
2. hop日志，需要跟进一下杨乐那块改造的进度，整体改造
3. 运维有个运维工具，如果不启动的话，会导致apiserver超时，大数据的任务可能会超时报错    ****
```
```Liujh
1.JDK升级测试覆盖度不够  -- 清理类的任务还不够
2. Job类型退出是怎么处理的，没听明白改造方案以及怎么生效的
3. how about crd？  ---  待定 ，需要go实现一部分
4. informer使用一个namespace还是anyNamespace，这块需要确认下
5. opencode可以尝试下
```
```Liurx
1. 服务改造，去掉了zk，把redis当成统一的服务监听入口  --- 在redis里跑了lua脚本
2. aws sdk升级在遇到了obs的签名问题后，回退到了1.0.1
3. datafusion的http以及增加了use支持
```