### 1. 参数解析

见总体架构



### 2. 创建执行器示例

```python
def load_executors(path: str) -> tp.List[exectools.Executor]:
    """Load hosts list from json file and return list of executors."""
    with open(path, 'r', encoding='utf-8') as f:
        hosts = json.load(f)

        exs = []
        for h in hosts:
            if h['type'] == 'local':
                ex = exectools.LocalExecutor()
            elif h['type'] == 'remote':
                ex = exectools.RemoteExecutor(h['host'], h['workdir'])
                if 'ssh_args' in h:
                    ex.ssh_extra_args += h['ssh_args']
                if 'scp_args' in h:
                    ex.scp_extra_args += h['scp_args']
            else:
                raise RuntimeError('invalid host type "' + h['type'] + '"')
            ex.ip = h['ip']
            exs.append(ex)
    return exs
```

从 JSON 格式的主机配置文件中解析主机信息，根据主机类型（本地或远程）初始化相应的执行器，最终返回这些执行器的列表。执行器（`Executor`）是框架中负责在特定主机上执行命令、管理资源的组件，支持本地和远程主机的操作。

### **代码解析**

1. **函数定义与参数**

   ```python
   def load_executors(path: str) -> tp.List[exectools.Executor]:
   ```

   - 输入参数 `path`：JSON 主机配置文件的路径。
   - 返回值：`exectools.Executor` 实例的列表，每个实例对应一台主机的执行器。

2. **加载并解析 JSON 文件**

   ```python
   with open(path, 'r', encoding='utf-8') as f:
       hosts = json.load(f)
   ```

   - 使用 `open` 打开 JSON 文件，通过 `json.load` 解析文件内容，得到一个包含主机配置的列表 `hosts`（每个元素是一台主机的配置字典）。

3. **遍历主机配置并创建执行器**

   ```python
   exs = []
   for h in hosts:
       # 根据主机类型创建对应执行器
       if h['type'] == 'local':
           ex = exectools.LocalExecutor()  # 本地执行器
       elif h['type'] == 'remote':
           # 远程执行器，需指定远程主机地址和工作目录
           ex = exectools.RemoteExecutor(h['host'], h['workdir'])
           # 若配置了额外的 SSH 参数，添加到执行器
           if 'ssh_args' in h:
               ex.ssh_extra_args += h['ssh_args']
           # 若配置了额外的 SCP 参数，添加到执行器
           if 'scp_args' in h:
               ex.scp_extra_args += h['scp_args']
       else:
           # 不支持的主机类型，抛出异常
           raise RuntimeError('invalid host type "' + h['type'] + '"')
       # 设置执行器对应的主机 IP 地址
       ex.ip = h['ip']
       exs.append(ex)
   ```

   - 遍历 hosts 列表中的每台主机配置：
     - 若主机类型为 `local`，创建 `LocalExecutor`（用于本地主机执行命令）。
     - 若主机类型为 `remote`，创建 `RemoteExecutor`（用于通过 SSH/SCP 操作远程主机），并可附加额外的 SSH/SCP 命令参数（如端口、认证方式等）。
     - 若主机类型未知，抛出 `RuntimeError` 异常。
   - 为每个执行器设置 `ip` 属性（从配置中读取），用于标识主机网络地址。

4. **返回执行器列表**

   ```python
   return exs
   ```

   - 将创建的所有执行器实例放入列表 `exs` 并返回，供后续分布式运行时（如 `DistributedSimpleRuntime`）使用。

### **总结**

`load_executors` 函数是 SimBricks 框架分布式运行的关键组件，它通过解析 JSON 配置文件，将不同类型（本地 / 远程）的主机转换为统一的执行器接口，使得框架能够透明地在多台主机上调度和执行实验任务。远程主机的 SSH/SCP 参数支持，也为跨节点通信提供了灵活性。



### 3. 添加运行实例

```python
def add_exp(
    e: exps.Experiment,
    rt: runtime.Runtime,
    run: int,
    prereq: tp.Optional[runtime.Run],
    create_cp: bool,
    restore_cp: bool,
    no_simbricks: bool,
    args: argparse.Namespace
):
    outpath = f'{args.outdir}/{e.name}-{run}.json'
    if os.path.exists(outpath) and not args.force:
        print(f'skip {e.name} run {run}')
        return None

    workdir = f'{args.workdir}/{e.name}/{run}'
    cpdir = f'{args.cpdir}/{e.name}/0'
    if args.shmdir is not None:
        shmdir = f'{args.shmdir}/{e.name}/{run}'

    env = experiment_environment.ExpEnv(args.repo, workdir, cpdir)
    env.create_cp = create_cp
    env.restore_cp = restore_cp
    env.no_simbricks = no_simbricks
    env.pcap_file = ''
    if args.pcap:
        env.pcap_file = workdir + '/pcap'
    if args.shmdir is not None:
        env.shm_base = os.path.abspath(shmdir)

    run = runtime.Run(e, run, env, outpath, prereq)
    rt.add_run(run)
    return run
```

### **参数说明**

- `e: exps.Experiment`：实验对象（包含实验的核心定义和配置）。
- `rt: runtime.Runtime`：运行时实例（负责调度和执行实验）。
- `run: int`：当前运行的编号（用于区分多次重复运行）。
- `prereq: tp.Optional[runtime.Run]`：前置依赖的运行实例（如检查点创建任务，可为 `None`）。
- `create_cp: bool`：是否创建检查点（checkpoint）。
- `restore_cp: bool`：是否从检查点恢复实验。
- `no_simbricks: bool`：实验是否不使用 SimBricks 框架组件。
- `args: argparse.Namespace`：解析后的命令行参数（包含路径、开关等配置）。

### **代码逻辑**

1. **输出路径检查与跳过逻辑**

   ```python
   outpath = f'{args.outdir}/{e.name}-{run}.json'
   if os.path.exists(outpath) and not args.force:
       print(f'skip {e.name} run {run}')
       return None
   ```

   - 生成当前实验运行的输出文件路径（`outpath`），格式为 `输出目录/实验名-运行编号.json`。
   - 若输出文件已存在且未指定 `--force`（强制重跑），则跳过该次运行并返回 `None`。

2. **目录路径设置**

   ```python
   workdir = f'{args.workdir}/{e.name}/{run}'  # 工作目录（临时文件）
   cpdir = f'{args.cpdir}/{e.name}/0'          # 检查点目录（固定为第 0 次运行的路径）
   if args.shmdir is not None:
       shmdir = f'{args.shmdir}/{e.name}/{run}'  # 共享内存目录（若指定）
   ```

   - 为每个实验的每次运行创建独立的工作目录（按 “实验名 / 运行编号” 分层）。
   - 检查点目录固定使用第 0 次运行的路径（确保多次运行可共享同一检查点）。

3. **实验环境配置（`ExpEnv`）**

   ```python
   env = experiment_environment.ExpEnv(args.repo, workdir, cpdir)
   env.create_cp = create_cp          # 是否创建检查点
   env.restore_cp = restore_cp        # 是否从检查点恢复
   env.no_simbricks = no_simbricks    # 是否禁用 SimBricks 组件
   env.pcap_file = ''
   if args.pcap:
       env.pcap_file = workdir + '/pcap'  # 启用 PCAP 时设置数据包捕获路径
   if args.shmdir is not None:
       env.shm_base = os.path.abspath(shmdir)  # 共享内存目录绝对路径
   ```

   - 初始化实验环境对象 `ExpEnv`，包含仓库路径、工作目录、检查点目录等基础信息。
   - 根据命令行参数（如 `--pcap`、`--shmdir`）和函数参数（如 `create_cp`）配置环境细节。

4. **创建并添加运行实例**

   ```python
   run = runtime.Run(e, run, env, outpath, prereq)
   rt.add_run(run)
   return run
   ```

   - 实例化 `runtime.Run` 对象，封装实验、运行编号、环境、输出路径和前置依赖。
   - 将该运行实例添加到运行时（`rt.add_run`），由运行时负责后续调度执行。
   - 返回创建的运行实例（供依赖链构建使用，如检查点任务作为后续任务的前置依赖）。

### **核心作用**

`add_exp` 是连接实验定义与运行时调度的关键函数：

- 它根据命令行参数和实验配置，自动生成必要的目录结构和环境参数。
- 通过检查输出文件是否存在，避免重复运行（除非强制重跑），提高效率。
- 支持检查点、PCAP 捕获、共享内存等高级功能的配置，为实验运行提供灵活的环境控制。
- 最终将配置好的运行实例提交给运行时，确保实验按预期被调度执行。



### 4. 主程序——main

1. 解析命令行参数与初始化执行器

```python
args = parse_args()
if args.hosts is None:
    executors = [exectools.LocalExecutor()]
else:
    executors = load_executors(args.hosts)
```

- 调用 `parse_args()` 解析用户输入的命令行参数（如实验模块路径、运行时类型、输出目录等）。
- 初始化执行器（executors）：
  - 若未指定 `--hosts`（主机配置文件），默认使用**本地执行器**（`LocalExecutor`）。
  - 若指定了 `--hosts`，通过 `load_executors` 从 JSON 文件加载主机配置，创建本地 / 远程执行器列表（支持多主机分布式执行）。

2. 初始化运行时（Runtime）

```python
if args.runtime == 'parallel':
    warn_multi_exec(executors)
    rt = runtime.LocalParallelRuntime(...)  # 本地并行运行时
elif args.runtime == 'slurm':
    rt = runtime.SlurmRuntime(...)  # 基于 Slurm 集群的运行时
elif args.runtime == 'dist':
    rt = runtime.DistributedSimpleRuntime(...)  # 分布式运行时
else:
    warn_multi_exec(executors)
    rt = runtime.LocalSimpleRuntime(...)  # 本地串行运行时
```

- **本地串行（默认）**：`LocalSimpleRuntime`，单线程依次执行实验。
- **本地并行**：`LocalParallelRuntime`，利用多核心并行执行实验（受 `--cores` 和 `--mem` 限制）。
- **Slurm 集群**：`SlurmRuntime`，适用于集群环境，通过 Slurm 调度任务。
- **分布式**：`DistributedSimpleRuntime`，在多台主机（通过 `--hosts` 配置）上分布式执行。
- 辅助逻辑：`warn_multi_exec` 在多执行器场景下提示部分运行时仅使用第一台主机。



3. 启用性能分析（可选）

```python
if args.profile_int:
    rt.enable_profiler(args.profile_int)
```

- 若指定 `--profile-int S`，则每 `S` 秒向模拟器发送 `SIGUSR1` 信号，用于性能分析（如采样程序运行状态）。



4. 加载实验定义

​	4.1 从 Python 模块加载（默认）

```python
experiments = []
for path in args.experiments:
    # 动态导入实验模块
    modname, _ = os.path.splitext(os.path.basename(path))
    spec = importlib.util.spec_from_file_location(modname, path)
    mod = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(mod)
    experiments += mod.experiments  # 收集模块中定义的实验
```

- 遍历 `args.experiments` 中的 Python 文件路径，动态导入模块并收集其中的 `experiments` 列表（实验定义）。
- 若指定 `--list`，仅打印所有实验名称并退出：

​	4.2 从 Pickle 文件加载（--pickled 模式）

```python
for path in args.experiments:
    with open(path, 'rb') as f:
        rt.add_run(pickle.load(f))  # 直接加载序列化的运行实例
```

- 适用于复用已保存的实验运行实例（如断点续跑），通过 `pickle` 反序列化并添加到运行时。



5. 实验预处理与添加到运行时

对加载的实验进行过滤、分布式转换、检查点配置后，添加到运行时：

​	5.1 动分布式转换（--auto-dist）

```python
if args.auto_dist and not isinstance(e, exps.DistributedExperiment):
    e = runtime.auto_dist(e, executors, args.proxy_type)
```

- 若实验不是分布式的，但指定了 `--auto-dist`，则自动将其转换为分布式实验，使用 `executors` 中的主机和指定的代理类型（`--proxy-type`，如 `sockets` 或 `rdma`）。

​	5.2 实验过滤（--filter）

```python
if args.filter and len(args.filter) > 0:
    match = False
    for f in args.filter:
        if fnmatch.fnmatch(e.name, f):  # 匹配 Unix 风格的模式（如 *.bench）
            match = True
            break
    if not match:
        continue  # 跳过不匹配的实验
```

- 仅保留名称匹配 `--filter` 模式的实验，实现实验的批量筛选。

​	5.3 检查点配置（e.checkpoint）

```python
if e.checkpoint:
    # 先添加一个创建检查点的运行（run=0，作为后续运行的依赖）
    prereq = add_exp(e, rt, 0, None, True, False, no_simbricks, args)
else:
    prereq = None  # 无检查点，无前置依赖
```

- 若实验需要检查点（`e.checkpoint=True`），先创建一个特殊的运行（`run=0`）用于生成检查点，后续运行将依赖该检查点（通过 `prereq` 参数）。

​	5.4 添加实验运行实例

```python
for run in range(args.firstrun, args.firstrun + args.runs):
    add_exp(e, rt, run, prereq, False, e.checkpoint, no_simbricks, args)
```

- 根据 `--runs`（重复次数）和 `--firstrun`（起始编号），为每个运行编号创建实验实例，通过 `add_exp` 函数添加到运行时。



6. 注册中断处理与启动运行时

```python
# 注册 SIGINT 信号处理（如 Ctrl+C），中断运行时
signal.signal(signal.SIGINT, lambda *_: rt.interrupt())

# 启动运行时，异步执行所有实验
asyncio.run(rt.start())
```

- 确保用户中断程序时，运行时能优雅退出（如清理资源）。
- 通过 `asyncio.run` 启动运行时的异步任务，开始执行所有添加的实验。