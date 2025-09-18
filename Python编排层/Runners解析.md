代码位置 experiment/simbricks/orchestration/runners.py

**Run：在调度层面描述一个实验，Runner：具体执行一个实验，Experiment：一个实验的具体配置（简单理解为实验中要启动的所有 Simulator 的集合），Nodeconfig：实验中一个节点的配置，Simulator：一个具体的节点配置对应模拟器的一次具体启动**



### ExperimentBaseRunner 类：一次实验执行的通用流程，包括模拟器依赖管理、启动、等待、清理等核心逻辑

#### 核心属性

| 属性名        | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| `exp`         | 待执行的实验对象（`Experiment` 或 `DistributedExperiment`）。 |
| `env`         | 实验环境配置（`ExpEnv` 实例），管理路径、工具路径等。        |
| `verbose`     | 是否启用详细日志输出（布尔值）。                             |
| `profile_int` | 性能分析间隔时间（秒），用于定期向模拟器发送 `SIGUSR1` 信号。 |
| `out`         | 实验输出管理器（`ExpOutput` 实例），记录执行结果、日志等。   |
| `running`     | 正在运行的模拟器列表，存储 `(Simulator, SimpleComponent)` 元组。 |
| `sockets`     | 需要清理的套接字路径列表，存储 `(Executor, 路径)` 元组。     |
| `wait_sims`   | 需要等待终止的模拟器组件（`Component` 实例）。               |

#### **核心方法**

##### 1. 抽象方法（由子类实现）

- `sim_executor(self, sim: Simulator) -> Executor`
  抽象方法，返回指定模拟器 `sim` 对应的执行器（`Executor`）。
  - 子类需根据实验类型（单机 / 分布式）决定执行器分配逻辑（如单机用同一个执行器，分布式按主机映射分配）。

##### 2. 模拟器依赖关系处理

- `sim_graph(self) -> tp.Dict[Simulator, tp.Set[Simulator]]`
  构建模拟器之间的依赖关系图，用于按拓扑顺序启动模拟器。
  - 依赖来源：每个模拟器的 `dependencies()` 方法（默认依赖）和 `extra_deps` 属性（额外依赖）。
  - 返回值：字典 `{模拟器: 依赖的模拟器集合}`，用于后续拓扑排序。

##### 3. 模拟器启动流程

- `async def start_sim(self, sim: Simulator) -> None`
  启动单个模拟器并等待其准备就绪，步骤包括：
  1. 生成模拟器启动命令：调用 `sim.run_cmd(self.env)` 获取具体命令。
  2. 创建执行组件：通过 `sim_executor` 获取执行器，创建 `SimpleComponent` 执行命令。
  3. 启动并监控：执行启动命令，将模拟器和组件加入 `running` 列表。
  4. 清理准备：记录模拟器需要清理的套接字路径（`sockets_cleanup`）。
  5. 等待就绪：等待模拟器的关键套接字创建（`sockets_wait`），确保启动完成。
  6. 延迟处理：若模拟器有启动延迟（`start_delay`），等待指定时间。
  7. 终止等待：若模拟器需要等待终止（`wait_terminate`），加入 `wait_sims` 列表。

##### 4. 实验准备阶段

- `async def prepare(self) -> None`
  实验启动前的准备工作，并行执行以提高效率：
  1. 生成配置文件：为所有主机模拟器生成配置 tar 包（`cfgtar_path`），并发送到执行器（`send_file`）。
  2. 执行预处理命令：调用每个模拟器的 `prep_cmds` 方法获取预处理命令，通过执行器并行执行。

##### 5. 实验执行与监控

- `async def run(self) -> ExpOutput`
  实验执行的主逻辑，协调模拟器启动、运行、终止全流程：
  1. 初始化输出：记录实验开始时间（`out.set_start`）。
  2. 拓扑排序：基于 `sim_graph` 对模拟器进行拓扑排序，确保按依赖顺序启动。
  3. 并行启动：按拓扑顺序分组启动就绪的模拟器（`start_sim`），每组内并行执行。
  4. 性能分析（可选）：若设置 `profile_int`，启动定时任务向所有运行中模拟器发送 `SIGUSR1`。
  5. 等待终止：调用 `wait_for_sims` 等待需要监控的模拟器终止。
  6. 异常处理：
     - 若被中断（`CancelledError`），标记实验为 “被中断”。
     - 若发生其他异常，标记实验为 “失败” 并打印堆栈。
  7. 清理与收集：调用 `terminate_collect_sims` 终止所有模拟器，收集输出并返回。

##### 6. 模拟器终止与结果收集

- `async def wait_for_sims(self) -> None`
  等待 `wait_sims` 列表中的模拟器组件终止（通常是主机模拟器，需等待其完成任务）。
- `async def terminate_collect_sims(self) -> ExpOutput`
  终止所有模拟器并收集执行结果：
  1. 记录结束时间：调用 `out.set_end`。
  2. 前置清理：执行 `before_cleanup`（子类可扩展）。
  3. 终止进程：对所有运行中的模拟器组件发送中断、终止、杀死信号（`int_term_kill`），确保停止。
  4. 清理套接字：删除 `sockets` 列表中记录的套接字路径。
  5. 收集输出：将每个模拟器的命令、 stdout、stderr 等信息添加到 `out`。
  6. 后置清理：执行 `after_cleanup`（子类可扩展）。

##### 7. 扩展点方法（空实现，供子类重写）

- `async def before_wait(self) -> None`：`wait_for_sims` 前执行的逻辑。
- `async def before_cleanup(self) -> None`：清理前执行的逻辑。
- `async def after_cleanup(self) -> None`：清理后执行的逻辑。



#### **与其他模块的交互**

- 依赖 `ExpEnv` 提供路径和环境配置。
- 依赖 `Executor` 执行命令。
- 依赖 `Simulator` 子类提供启动命令、依赖关系、清理路径等信息。
- 结果通过 `ExpOutput` 收集并输出到文件。
