[github](https://github.com/potherb99/MethodProbe)
# 前言:这不是替代,是互补

先说结论:**MethodProbe 不是为了替代 Arthas**,两者是不同场景下的最优解。

如果把 Java 诊断工具比作医疗设备:

- • **Arthas** = 全能 CT 机,功能强大,随时可以深度检查
    
- • **MethodProbe** = 智能手环,长期佩戴,持续监控健康指标
    

---

# 一、核心定位差异

### Arthas:交互式诊断工具

Arthas 是一个**命令行交互式**的诊断工具,设计理念是:

> 当问题发生时,**人工介入**,通过命令精准排查

**典型使用流程:**

`# 1. Attach 到目标进程   java -jar arthas-boot.jar      # 2. 找到目标类   sc -d com.example.Service      # 3. Trace 方法调用   trace com.example.Service handleRequest      # 4. 观察方法参数   watch com.example.Service handleRequest '{params, returnObj}' -x 2      # 5. 问题解决后退出   quit`

### MethodProbe:持续性监控工具

MethodProbe 是一个**配置驱动的被动式**监控工具,设计理念是:

> 在应用启动时嵌入,**自动捕获**超时/异常,无需人工干预

**典型使用流程:**

`# 1. 启动时加载 Agent   java -javaagent:methodprobe.jar=config=agent.properties -jar app.jar      # 2. 配置文件定义监控规则   probe.flat.packages=com.example   probe.flat.threshold=100   probe.snapshot.enabled=true      # 3. 应用运行,自动记录超时/异常   # 4. 问题发生时,查看日志和快照文件   # 5. Agent 持续运行,无需退出`

---

# 二、功能对比矩阵

|维度|Arthas|MethodProbe|说明|
|---|---|---|---|
|**使用模式**|交互式,主动触发|配置式,被动捕获|Arthas 需要手敲命令,MethodProbe 启动后自动工作|
|**部署方式**|Attach 到运行中的进程|JVM 启动参数加载|Arthas 可以随时接入,MethodProbe 需要重启|
|**持续监控**|❌ 不适合|✅ 核心场景|Arthas 会影响性能,不能长期开启|
|**自动捕获**|❌ 需手动触发|✅ 自动记录|MethodProbe 超时/异常自动保存快照|
|**数据持久化**|❌ 控制台输出|✅ 文件+快照|MethodProbe 支持日志滚动和快照自动清理|
|**方法调用树**|✅ trace 命令|✅ Tree 模式|两者都支持,但触发方式不同|
|**参数快照**|✅ tt/watch 命令|✅ Snapshot 功能|Arthas 更灵活,MethodProbe 更自动化|

---

# 三、典型场景对比

### 场景 1:定位偶发性能问题

**问题描述:**  
生产环境某接口偶尔会慢到 3 秒,一天出现 2-3 次,无规律。

#### 用 Arthas:

`# 难点:不知道什么时候会慢,需要一直守着   # 1. SSH 登录服务器   # 2. Attach Arthas   # 3. 执行 trace 命令   trace com.example.Controller handleRequest   # 4. 等待问题复现...等了 1 小时还没出现   # 5. 不能一直开着(影响性能),只能先退出`

**结果:**  无法捕获,除非 7x24 小时守着

#### 用 MethodProbe:

`# 启动时配置,自动捕获   probe.tree.enabled=true   probe.tree.entry.methods=com.example.Controller.handleRequest   probe.tree.threshold=500  # 超过 500ms 自动记录   probe.snapshot.enabled=true`

**结果:**  问题发生时自动记录调用树和快照,第二天查日志即可

---

### 场景 2:生产环境异常追踪

**问题描述:**  
某个方法偶尔抛 `NullPointerException`,但不知道是什么入参导致的。

#### 用 Arthas:

`# 需要提前埋点,且不能长期开启   watch com.example.Service process '{params}' -e   # -e 表示只在抛异常时输出`

**问题:**

1. 1. 需要**提前知道**要监控哪个方法
    
2. 2. watch 命令开销大,不能长期开启
    
3. 3. 问题复现时你可能已经退出了
    

#### 用 MethodProbe:

`# 启动时配置,自动捕获所有异常   probe.flat.enabled=true   probe.flat.packages=com.example   probe.flat.trigger=exception   probe.snapshot.enabled=true   probe.snapshot.trigger=exception`

**结果:**  任何异常发生时自动保存快照,包含入参、堆栈、耗时

---

### 场景 3:临时排查未知问题

**问题描述:**  
突然收到告警,RT 飙升,但不知道是哪个方法慢,需要快速定位。

#### 用 Arthas:

`# 快速介入,实时诊断   # 1. Attach 进程   java -jar arthas-boot.jar      # 2. 快速找到慢方法   dashboard  # 查看线程、内存   thread -n 3  # 查看 CPU 占用最高的 3 个线程   trace com.example.Controller *  # Trace 所有方法`

**结果:**  Arthas 胜出,响应速度快

#### 用 MethodProbe:

`# 通过HTTP 接口,可以动态添加监控点，但是知道是那个位置。   curl -X POST http://localhost:9876/flat/package/add -d "packageName=com.example"`

**结果:**  MethodProbe 需要提前规划,临时排查不如 Arthas 灵活

---

# 四、总结

### MethodProbe 的价值

MethodProbe 填补了 Arthas 的空白:

|场景|MethodProbe 的优势|
|---|---|
|**持续监控**|7x24 小时自动记录超时/异常,无需人工守着|
|**偶发问题**|自动捕获快照,事后分析,不错过任何线索|
|**生产环境**|极低开销(< 1%),可以长期运行|
|**团队协作**|配置文件可版本管理,新人上手快|
|**数据持久化**|日志滚动、快照自动清理,便于回溯|

### Arthas 适合:

- • ✅ **临时排查**:问题正在发生,需要立即介入
    
- • ✅ **深度诊断**:需要观察返回值、字段、堆栈等详细信息
    
- • ✅ **探索式调试**:不确定问题在哪,需要四处查看
    
- • ✅ **学习研究**:研究框架源码、JVM 行为
    

### MethodProbe 适合:

- • ✅ **持续监控**:7x24 小时自动记录超时/异常
    
- • ✅ **偶发问题**:无规律的性能抖动、随机异常
    
- • ✅ **生产环境**:低开销,可以长期运行
    
- • ✅ **团队协作**:配置驱动,降低排查门槛