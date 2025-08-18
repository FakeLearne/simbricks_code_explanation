文件：docker/simbricks-run

```bash
#!/bin/bash
exec python3 /simbricks/experiments/run.py "$@"
```

推测 simbricks-run 程序本质上就是执行 python 脚本 run.py



![image-20250818143541828](C:\Users\cheny\AppData\Roaming\Typora\typora-user-images\image-20250818143541828.png)

大小仅为 60 字节





```shell
root@359dc86d967f:/simbricks/images/output-base# simbricks-run --help
usage: run.py [-h] [--list] [--filter PATTERN [PATTERN ...]] [--pickled] [--runs N] [--firstrun N] [--force] [--verbose] [--pcap] [--profile-int S] [--repo DIR] [--workdir DIR] [--outdir DIR] [--cpdir DIR]
              [--hosts JSON_FILE] [--shmdir DIR] [--parallel] [--cores N] [--mem N] [--slurm] [--slurmdir DIR] [--dist] [--auto-dist] [--proxy-type TYPE]
              EXP [EXP ...]

positional arguments:
  EXP                   Python modules to load the experiments from

options:
  -h, --help            show this help message and exit
  --list                List available experiment names
  --filter PATTERN [PATTERN ...]
                        Only run experiments matching the given Unix shell style patterns
  --pickled             Interpret experiment modules as pickled runs instead of .py files
  --runs N              Number of repetition of each experiment
  --firstrun N          ID for first run
  --force               Run experiments even if output already exists (overwrites output)
  --verbose             Verbose output, for example, print component simulators' output
  --pcap                Dump pcap file (if supported by component simulator)
  --profile-int S       Enable periodic sigusr1 to each simulator every S seconds.

Environment:
  --repo DIR            SimBricks repository directory
  --workdir DIR         Work directory base
  --outdir DIR          Output directory base
  --cpdir DIR           Checkpoint directory base
  --hosts JSON_FILE     List of hosts to use (json)
  --shmdir DIR          Shared memory directory base (workdir if not set)

Parallel Runtime:
  --parallel            Use parallel instead of sequential runtime
  --cores N             Number of cores to use for parallel runs
  --mem N               Memory limit for parallel runs (in MB)

Slurm Runtime:
  --slurm               Use slurm instead of sequential runtime
  --slurmdir DIR        Slurm communication directory

Distributed Runtime:
  --dist                Use sequential distributed runtime instead of local
  --auto-dist           Automatically distribute non-distributed experiments
  --proxy-type TYPE     Proxy type to use (sockets,rdma) for auto distribution
```

项目中对 simbricks-run 程序的解释，参数列表中的定义都可以在 run.py 中找到

解释：

#### 1. **通用实验参数**（无分组，直接添加到主解析器）

- `experiments`（必选参数）：
  位置参数，接受一个或多个字符串，表示包含实验定义的 Python 模块路径（`.py` 文件）。例如 `python run.py my_experiment.py` 中，`my_experiment.py` 即为此参数。
- `--list`：
  布尔开关，若指定，仅列出所有可用实验的名称，不实际运行。
- `--filter`：
  接受一个或多个 Unix 风格的通配符模式（如 `netperf*`），仅运行名称匹配模式的实验。
- `--pickled`：
  布尔开关，默认 `False`。若指定，将输入的 `experiments` 解析为序列化（pickled）的运行对象文件，而非 Python 模块。
- `--runs`：
  整数，默认 `1`，指定每个实验的重复运行次数。
- `--firstrun`：
  整数，默认 `1`，指定首次运行的 ID（用于区分多次重复运行的输出目录）。
- `--force`：
  布尔开关，默认 `False`。若指定，即使实验输出已存在也会强制重新运行（覆盖原有输出）。
- `--verbose`：
  布尔开关，默认 `False`。启用后输出详细日志（如各模拟器组件的运行输出）。
- `--pcap`：
  布尔开关，默认 `False`。若指定，支持的模拟器组件会生成 PCAP 数据包捕获文件。`（pcap 是数据包捕获）`

- `--profile-int`：
  整数（单位：秒），默认 `None`。若指定，将定期向每个模拟器发送 `SIGUSR1` 信号（用于性能分析）。

#### 2. **环境参数**（`Environment` 分组）

用于配置实验运行的目录、路径等环境信息：

- `--repo`：
  字符串，默认值为脚本所在目录的父目录，指定 SimBricks 仓库的根目录。
- `--workdir`：
  字符串，默认 `./out/`，指定工作目录的基础路径（用于存放临时文件）。
- `--outdir`：
  字符串，默认 `./out/`，指定输出目录的基础路径（用于存放实验结果）。
- `--cpdir`：
  字符串，默认 `./out/`，指定检查点（checkpoint）目录的基础路径（用于保存 / 恢复实验状态）。
- `--hosts`：
  字符串，默认 `None`，指定包含主机列表的 JSON 文件路径（用于分布式运行）。
- `--shmdir`：
  字符串，默认 `None`，指定共享内存目录的基础路径（未指定时使用 `workdir`）。

#### 3. **并行运行时参数**（`Parallel Runtime` 分组）

用于配置本地并行执行实验的参数：

- `--parallel`：
  布尔开关，指定后将运行时设置为 `parallel`（默认 `sequential` 串行），即本地并行执行实验。
- `--cores`：
  整数，默认值为当前进程可使用的 CPU 核心数，指定并行运行时可使用的核心数量。
- `--mem`：
  整数（单位：MB），默认 `None`，指定并行运行时的内存限制。

---

​	**下面这里的集群和分布式指的是使用多台主机共同执行 simbricks 仿真程序，而不是指一个 simbricks 仿真中模拟多 hosts 通信的场景**

#### 4. **Slurm 运行时参数**（`Slurm Runtime` 分组）

用于配置通过 Slurm 调度系统执行实验的参数（适用于集群环境）：

- `--slurm`：
  布尔开关，指定后将运行时设置为 `slurm`（默认 `sequential`），即通过 Slurm 调度实验。
- `--slurmdir`：
  字符串，默认 `./slurm/`，指定 Slurm 相关文件（如调度脚本、日志）的存放目录。

#### 5. **分布式运行时参数**（`Distributed Runtime` 分组）

用于配置分布式执行实验的参数（跨多台主机）：

- `--dist`：
  布尔开关，指定后将运行时设置为 `dist`（默认 `sequential`），即使用分布式运行时。
- `--auto-dist`：
  布尔开关，默认 `False`。若指定，会自动将非分布式实验转换为分布式（基于 `--hosts` 配置）。
- `--proxy-type`：
  字符串，默认 `sockets`，指定自动分布式时使用的代理类型（`sockets` 或 `rdma`）。