代码位置 experiment/simbricks/orchestration/runtime/*

定义了实验运行的核心抽象类和数据结构



两大核心类：**Run 类，Runtime 类**



### Run 类：单个实验运行的封装（Defines a single execution run for an experiment）

`Run` 类用于描述**一次具体的实验执行实例**，包含实验本身、运行索引、环境配置等关键信息，是连接**实验定义与执行逻辑**的桥梁。

##### 1. 主要属性

| 属性名       | 作用描述                                                     |
| ------------ | ------------------------------------------------------------ |
| `experiment` | 关联的实验对象（`Experiment` 实例），包含实验的核心配置和逻辑 |
| `index`      | 运行索引（用于区分同一实验的多次重复运行）                   |
| `env`        | 实验环境配置（`ExpEnv` 实例），包含工作目录、输出路径等环境参数 |
| `outpath`    | 输出结果文件路径                                             |
| `output`     | 实验执行后的输出数据（`ExpOutput` 实例），运行结束后填充     |
| `prereq`     | 前置依赖的 `Run` 实例（用于控制运行顺序，如 A 运行完成后再运行 B） |
| `job_id`     | Slurm 环境下的作业 ID（仅在使用 Slurm 运行时有效）           |

##### 2. 核心方法

- `name(self) -> str`：返回运行的唯一标识名称，格式为 `[实验名].[索引]`（如 `test_exp.1`），用于日志和输出文件命名。
- `prep_dirs(self, executor=LocalExecutor()) -> None`：准备实验运行所需的目录，包括：
  - 清理并重建工作目录（`workdir`）、共享内存目录（`shm_base`）、检查点目录（`cpdir`）
  - 同时调用执行器（`executor`）的目录操作方法，确保本地或远程环境的目录一致性
  - 作用：避免残留文件影响实验结果，保证每次运行的环境干净





### Runtime 类：多个 Run 示例的封装与管理（Base class for managing the execution of multiple runs）

`Runtime` 是所有运行时管理器（如本地顺序执行、本地并行执行、Slurm 集群执行等）的基类，定义了统一的接口规范。

##### 1. 主要属性

- `_interrupted`：布尔值，标记是否收到中断信号（如用户 Ctrl+C），用于安全终止运行中的实验。
- `profile_int`：性能分析间隔时间（秒），用于定期向模拟器发送信号进行性能采样（可选功能）。

##### 2. 核心方法（抽象方法，需子类实现）

| 方法名               | 作用描述                                                     |
| -------------------- | ------------------------------------------------------------ |
| `add_run(self, run)` | 向运行时添加一个 `Run` 实例，用于后续执行                    |
| `start(self)`        | 异步启动所有添加的 `Run`，按特定逻辑（顺序、并行、分布式等）执行实验 |
| `interrupt_handler`  | 中断信号处理函数，负责优雅停止当前运行的实验并收集输出       |

##### 3. 通用方法

- `interrupt(self)`：触发中断流程，设置 `_interrupted` 标记并调用 `interrupt_handler`，确保只执行一次中断逻辑。
- `enable_profiler(self, profile_int)`：启用性能分析，设置采样间隔时间。





### LocalSimpleRuntime：本地顺序运行时（Execute runs locally in sequence）

##### 核心属性

- `runnable`：待执行的`Run`实例列表（每个`Run`对应一次实验运行）。
- `complete`：已完成的`Run`实例列表。
- `executor`：执行器（默认`LocalExecutor`），负责本地命令执行和文件操作。
- `_running`：当前正在执行的异步任务（`asyncio.Task`），用于中断控制。

##### 核心方法

- `add_run(self, run: Run)`：
  将实验运行实例（`Run`）添加到`runnable`列表，等待执行。
- `do_run(self, run: Run)`：
  实际执行单个实验的逻辑：
  1. 初始化`ExperimentSimpleRunner`（本地实验运行器），关联执行器和实验环境。（Run：在调度层面描述一个实验，Runner：具体执行一个实验）
  2. 调用`run.prep_dirs()`准备工作目录（清理旧文件、创建新目录）。
  3. 调用`runner.prepare()`完成实验前置准备（如生成配置文件、传输依赖）。
  4. 调用`runner.run()`执行实验，获取输出结果（`ExpOutput`）。
  5. 将结果写入 JSON 文件（`run.outpath`），并将`Run`移至`complete`列表。
- `start(self)`：
  按顺序执行`runnable`中的所有实验：
  - 遍历`runnable`列表，为每个`Run`创建异步任务（`_running`）。
  - 等待当前任务完成后，再执行下一个，确保顺序性。
  - 若收到中断信号（`_interrupted`），立即停止后续执行。
- `interrupt_handler(self)`：
  中断处理逻辑：取消当前正在执行的`_running`任务，实现实验的优雅停止。



### LocalParallelRuntime：本地并行运行时（Execute runs locally in parallel on multiple cores）

不过多赘述
