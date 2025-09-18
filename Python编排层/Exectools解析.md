定义了 SimBricks 框架中**进程管理与跨主机执行**的核心组件，主要负责在本地或远程主机上启动、监控、终止进程，以及处理文件传输、目录操作等基础任务。其核心是通过 “执行器（Executor）” 抽象，屏蔽本地与远程环境的差异，为上层实验调度提供统一的操作接口。



在 SimBricks 框架中，执行器（Executor）是连接**实验调度逻辑**与**物理执行环境**的 “桥梁”（**不准确**），其核心作用是：

#### 1. **环境适配与统一接口**

屏蔽本地与远程主机的差异，提供一致的操作接口（如创建进程、传输文件、管理目录）。上层逻辑（如 `DistributedSimpleRuntime` 运行时）无需关心具体是在本地还是远程执行，只需调用执行器的接口即可。

例如：

- 本地实验使用 `LocalExecutor`，直接操作本地文件系统和进程；
- 分布式实验使用 `RemoteExecutor`，通过 SSH/SCP 透明地操作远程主机，上层代码无需修改。

#### 2. **实验资源的准备与管理**

在实验启动前，执行器负责准备运行环境：

- 创建工作目录（`mkdir()`）、清理旧文件（`rmtree()`）；
- 传输实验配置文件（如虚拟机镜像、拓扑配置）到目标主机（`send_file()`）；
- 等待模拟器所需的套接字文件（如 `await_file()` 等待 NIC 与主机的通信套接字创建）。

#### 3. **进程生命周期的全托管**

实验中的模拟器（如 QEMU、gem5、NS3）均通过执行器启动为进程，并由执行器全程管理：

- 启动进程（`create_component().start()`）；
- 实时监控输出（`process_out`/`process_err`），便于调试；
- 接收中断信号（如 Ctrl+C）时，通过 `int_term_kill()` 优雅终止进程，避免资源泄漏；
- 分布式场景中，`RemoteExecutor` 通过 SSH 远程控制进程，确保多主机协同执行。

#### 4. **支持分布式实验的核心**

在多主机分布式实验中，执行器是实现跨机协同的关键：

- 框架通过 `--hosts` 配置文件加载多个 `RemoteExecutor`；
- 不同模拟器组件（如主机、NIC、网络）被分配到不同执行器（对应不同物理主机）；
- 执行器通过统一接口协调跨机文件传输、进程同步，确保分布式实验的一致性。





分为**组件（Component）** 和**执行器（Executor）** 两大体系：

#### 1. **Component 体系：进程生命周期管理**

**负责单个进程的启动、输入输出处理、信号发送（中断、终止等），是执行命令的最小单元。基于 python 的 asyncio，对异步启动子进程的过程做了一些特定的修改（主要是如何处理输出），来便于 Simbricks 管理**

1. #### `Component` 类（基础进程组件）

**角色**：所有进程组件的抽象基类，定义了进程生命周期管理的核心接口。
**核心功能**：

- 封装**进程启动、监控、终止的基础逻辑**
  - 进程的启动与终止
  - 标准输出（stdout）和标准错误（stderr）的实时捕获与解析
  - 信号发送（如中断、终止、杀死进程）
  - 异步等待进程结束
- 关键方法：
  - `__init__`：初始化进程参数（命令、工作目录、环境变量等）。
  - `start()`：异步启动进程，通过 `asyncio.create_subprocess_exec` 创建子进程，并绑定 stdout/stderr 流。
  - `_read_stream`：异步读取进程输出流（stdout/stderr），按行解析并触发回调。
  - `interrupt()`/`terminate()`/`kill()`：分别发送 SIGINT、SIGTERM、SIGKILL 信号，控制进程终止。
  - `wait()`：等待进程结束，返回退出码。
- 为所有本地进程（如 QEMU、gem5 模拟器）提供统一的生命周期管理接口，确保进程启动、监控、终止的一致性。

```python
    async def start(self) -> None:
        if self.with_stdin:
            stdin = asyncio.subprocess.PIPE
        else:
            stdin = asyncio.subprocess.DEVNULL

        self._proc = await asyncio.create_subprocess_exec( # create_subprocess_exec：异步启动外部命令
            *self.cmd_parts,
            stdout=asyncio.subprocess.PIPE,	# 捕获 stdout
            stderr=asyncio.subprocess.PIPE,	# 捕获 stderr
            stdin=stdin,
        )
        # 启动等待任务（监控进程结束和输出）
    	self._terminate_future = asyncio.create_task(self._waiter())
    	await self.started()  # 调用启动钩子
```

- 关键属性：

    | `cmd_parts`               | 进程启动命令的拆分列表（如 `['ls', '-l']`），用于创建子进程  |
    | ------------------------- | ------------------------------------------------------------ |
    | `with_stdin`              | 是否需要为进程提供标准输入（`stdin`），默认关闭（使用 `DEVNULL`） |
    | `stdout`/`stderr`         | 存储已解析的进程输出行（字符串列表）                         |
    | `stdout_buf`/`stderr_buf` | 未解析的输出缓冲区（字节数组），用于临时存储未换行的输出数据 |
    | **`_proc`**               | **异步子进程对象（`asyncio.subprocess.Process`），封装实际运行的进程** |
    | `_terminate_future`       | 异步任务（`asyncio.Task`），用于等待进程结束并处理输出       |

- 进程控制：信号发送与终止

  - `interrupt()`：发送 `SIGINT` 信号（对应 `Ctrl+C`）。

  - `terminate()`：发送 `SIGTERM` 信号（请求进程终止）。

  - `kill()`：发送 `SIGKILL` 信号（强制杀死进程，无法忽略）。

  - int_term_kill(delay=5)：渐进式终止策略：
    1. 先发送 `SIGINT`，等待 `delay` 秒；
    2. 若未结束，发送 `SIGTERM`，再等待 `delay` 秒；
    3. 若仍未结束，发送 `SIGKILL` 强制终止。
    
    

2. #### `SimpleComponent` 类

① **标准输出处理 `process_out`**

```python
async def process_out(self, lines: tp.List[str], eof: bool) -> None:
    if self.verbose:
        for _ in lines:
            print(self.label, 'OUT:', lines, flush=True)
```

- 重写父类钩子方法，当 verbose=True 时：
  - 打印每条标准输出行，格式为 `[label] OUT: [输出内容]`
  - 使用 `flush=True` 确保日志实时输出（不缓存）
- 注意：代码中存在一个小问题，`for _ in lines` 循环中实际打印的是整个 `lines` 列表，而非单个元素，可能导致重复输出（应为 `for line in lines: print(..., line, ...)`）？

② **标准错误处理 `process_err`**

```python
async def process_err(self, lines: tp.List[str], eof: bool) -> None:
    if self.verbose:
        for _ in lines:
            print(self.label, 'ERR:', lines, flush=True)
```

- 与 process_out 逻辑类似，处理标准错误输出：
  - 打印格式为 `[label] ERR: [错误内容]`

③ **进程终止处理 `terminated`**

```python
async def terminated(self, rc: int) -> None:
    if self.verbose:
        print(self.label, 'TERMINATED:', rc, flush=True)
    if not self.canfail and rc != 0:
        raise RuntimeError('Command Failed: ' + str(self.cmd_parts))
```

- 进程终止时触发，接收进程返回码 rc：
  - 若 `verbose=True`，打印终止信息（`[label] TERMINATED: [返回码]`）
  - 若 `canfail=False` 且返回码非 0（命令执行失败），抛出 `RuntimeError` 异常，中断流程
  - 若 `canfail=True`，即使命令失败也不抛异常，继续执行后续逻辑





#### 2. **Executor 体系：执行环境抽象**

定义了在特定环境（本地或远程）中执行操作的统一接口，其子类分别适配本地和远程主机。

1. #### **`Executor` 类**

`Executor` 继承自 `abc.ABC`（抽象基类），通过定义抽象方法强制子类实现特定功能，同时提供了两个通用的默认方法（`run_cmdlist` 和 `await_files`）。

- 抽象方法

| 方法名             | 作用描述                                                     |
| ------------------ | ------------------------------------------------------------ |
| `create_component` | 创建一个组件（`SimpleComponent` 或其子类），用于执行命令。参数包括标签（`label`）、命令拆分列表（`parts`）等。 |
| `await_file`       | 等待指定路径的文件出现，可设置检查间隔（`delay`）和是否打印日志（`verbose`）。 |
| `send_file`        | 将文件发送到目标环境（本地执行器无需操作，远程执行器通过 `scp` 实现）。 |
| `mkdir`            | 创建目录（支持递归创建父目录）。                             |
| `rmtree`           | 递归删除目录或文件（类似 `rm -rf`）。                        |

- `run_cmdlist`：批量执行命令列表

```python
async def run_cmdlist(self, label: str, cmds: tp.List[str], verbose=True) -> None:
    i = 0
    for cmd in cmds:
        # 为每个命令创建 component，标签格式为 "原始标签.序号"（如 "test.0"、"test.1"）
        cmd_c = self.create_component(
            label + '.' + str(i), shlex.split(cmd), verbose=verbose
        )
        await cmd_c.start()  # 启动命令
        await cmd_c.wait()   # 等待命令执行完成
        i += 1
```

- `await_files`：并行等待多个文件出现

```python
async def await_files(self, paths: tp.List[str], *args, **kwargs) -> None:
    xs = []
    for p in paths:
        # 为每个文件创建一个异步任务，等待其出现
        waiter = asyncio.create_task(self.await_file(p, *args, **kwargs))
        xs.append(waiter)
    # 等待所有任务完成（并行执行）
    await asyncio.gather(*xs)
```

2. #### **`LocalExecutor` 类**

在本地系统上完成以下操作：（都是很简单的逻辑）

- 创建 component
- 等待文件出现
- 创建/删除 目录/文件

