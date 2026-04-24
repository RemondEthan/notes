- [x] 通过swan task获取driver pod或者jm pod关系已经有了还是需要新增？ ✅ 2024-09-18
- [x] 是否已经验证 ✅ 2024-09-18
~~~plaintext
spark:driver-pod-id:driverUIport/api/v1/applications/spark-app-selector/allexecutors

flink:jobmanager-pod:8081/taskmanagers

如果上述的返回的资源列表>=1即表示资源已经分配,可讲对应的资源从swan-resourcemanager中的pending队列撤回。
~~~

**获取spark真实的任务状态**
