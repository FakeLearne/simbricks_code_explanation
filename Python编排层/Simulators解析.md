代码位置 experiment/simbricks/orchestration/simulators.py

该模块定义了各种模拟器的信息，包括如何封装、启动、配置，代码量较大，但主要是因为不同种类的模拟器很多，逻辑上没有很大难度。



### 1. Simulator 类

`Simulator` 类是 SimBricks 框架中所有模拟器的基类，定义了各类模拟器（主机、网卡、网络等）的通用接口和基础属性，是整个模拟器体系的抽象层。

#### **核心属性**

- `extra_deps: tp.List[Simulator]`
  存储当前模拟器的额外依赖项（其他模拟器），用于构建依赖关系图，确保按正确顺序启动。
- `name: str`
  模拟器的名称，用于标识和区分不同模拟器实例，需在实验中保持唯一（由子类或用户设置）。

#### **核心方法**

1. 资源需求计算

用于框架评估实验所需的系统资源，辅助调度和资源分配：

- `resreq_cores(self) -> int`
  返回模拟器所需的核心数，默认返回 `1`。子类可根据自身需求重写（如 `QemuHost` 根据是否同步或核心数调整）。
- `resreq_mem(self) -> int`
  返回模拟器所需的内存（MB），默认返回 `64`。子类可重写（如 `Gem5Host` 需要更大内存，返回 `4096`）。

2. 名称与标识

- `full_name(self) -> str`
  返回模拟器的完整名称，默认返回空字符串。子类需重写以区分不同类型的模拟器（如 `NICSim` 返回 `nic.{name}`，`HostSim` 返回 `host.{name}`）。

3. 执行准备与命令

- `prep_cmds(self, env: ExpEnv) -> tp.List[str]`
  返回执行模拟器前的准备命令列表（如创建目录、复制文件等），默认返回空列表。子类可重写（如 `QemuHost` 会生成镜像文件的准备命令）。
- `run_cmd(self, env: ExpEnv) -> tp.Optional[str]`
  返回启动模拟器的命令字符串，默认返回 `None`（表示 “虚拟模拟器”，无需实际启动）。子类必须重写以提供具体的启动命令（如 `Gem5Host` 生成 gem5 模拟器的启动命令）。

4. 依赖关系

- `dependencies(self) -> tp.List[Simulator]`
  返回当前模拟器依赖的其他模拟器列表，默认返回空列表。子类可重写以声明依赖（如 `HostSim` 依赖其连接的网卡和网络模拟器）。

5. 套接字管理

用于模拟器间通信的套接字文件管理，确保资源正确清理和启动就绪检查：

- `sockets_cleanup(self, env: ExpEnv) -> tp.List[str]`
  返回实验结束后需要清理的套接字文件路径列表，默认返回空列表。子类可重写（如 `NICSim` 需要清理 PCI 通信和以太网通信的套接字）。
- `sockets_wait(self, env: ExpEnv) -> tp.List[str]`
  返回模拟器启动后需要等待的套接字文件路径列表（用于确认模拟器已就绪），默认返回空列表。子类可重写（如 `NetSim` 等待网络监听套接字）。

6. 启动与执行控制

- `start_delay(self) -> int`
  返回模拟器启动后需要等待的秒数（确保依赖组件就绪），默认返回 `5`。子类可根据需要调整（如 `NetProxy` 设置为 `10`）。
- `wait_terminate(self) -> bool`
  指示框架是否需要等待该模拟器终止后再结束实验，默认返回 `False`。子类可重写（如 `HostSim` 通过 `wait` 属性控制是否等待）。



### 2. PCIDevSim 类

`PCIDevSim` 类是 SimBricks 框架中所有 PCIe 设备模拟器的基类，继承自 `Simulator` 类，定义了 PCIe 设备的通用属性和行为，为 NIC 等具体 PCIe 设备模拟器提供统一接口。

#### **核心属性**

| 属性名        | 类型 / 默认值 | 说明                                                         |
| ------------- | ------------- | ------------------------------------------------------------ |
| `sync_mode`   | `int`（0）    | **同步模式：0 表示非同步运行，模拟器进程会以最快的速度处理接收的消息，1 表示同步运行（部分模拟器可能支持更多模式），模拟器会在正确的时间戳才处理消息。Simbricks中默认设置为0，即非同步模式，此时模拟器只能做功能层面的验证，对于性能测试没有意义！** |
| `start_tick`  | `int`（0）    | 表示 PCIe 设备模拟器的**仿真启动时间戳**，用于控制设备模拟器开始参与仿真的时机。它的核心作用是在设备模拟器并非从仿真初始阶段就接入系统时，确保其能与已运行的关联模拟器（如主机、网络等）正确同步时间进度。 |
| `sync_period` | `int`（500）  | 同步消息发送周期（纳秒），控制向关联组件发送同步信息的频率。 |
| `pci_latency` | `int`（500）  | PCIe 通信延迟（纳秒），模拟设备与主机间通过 PCIe 传输的延迟。 |

#### **核心方法**

1. 名称标识

- `full_name(self) -> str`
  返回 PCIe 设备的完整名称，格式为 `dev.{name}`（例如 `dev.nic0`），用于区分不同类型的模拟器。

2. 设备类型判断

- `is_nic(self) -> bool`
  判断当前设备是否为网卡，默认返回 `False`。子类（如 `NICSim`）会重写此方法以标识自身为网卡类型。

3. 套接字管理

用于管理 PCIe 设备与主机 / 其他组件通信的套接字文件，确保资源清理和启动就绪检查：

- `sockets_cleanup(self, env: ExpEnv) -> tp.List[str]`
  返回实验结束后需要清理的套接字路径列表，包括 PCI 通信套接字（`env.dev_pci_path(self)`）和共享内存路径（`env.dev_shm_path(self)`）。
- `sockets_wait(self, env: ExpEnv) -> tp.List[str]`
  返回启动时需要等待的套接字路径列表（仅 PCI 通信套接字），确保设备模拟器就绪后再进行后续操作。



### 3. NICSim 类

`NICSim` 类是 SimBricks 框架中所有网卡模拟器的基类，继承自 `PCIDevSim`（PCIe 设备模拟器基类），专门定义了网卡模拟器的通用属性和行为，为各类具体网卡（如 Intel I40e、Corundum 系列）提供统一接口。

#### **核心属性**

| 属性名                    | 类型 / 默认值      | 说明                                                         |
| ------------------------- | ------------------ | ------------------------------------------------------------ |
| `network`                 | `NetSim`（`None`） | 关联的网络模拟器（如交换机、网桥等），通过 `set_network()` 方法绑定。 |
| `mac`                     | `str`（`None`）    | 网卡的 MAC 地址，用于以太网通信标识，格式通常为 `xx:xx:xx:xx:xx:xx`。 |
| `eth_latency`             | `int`（500）       | 以太网通信延迟（纳秒），模拟网卡与网络组件间的数据传输延迟。 |
| 继承自 `PCIDevSim` 的属性 | -                  | 包括 `sync_mode`（同步模式）、`sync_period`（同步周期）、`pci_latency`（PCIe 延迟）等，用于与主机的同步和通信控制。 |

#### **核心方法**

1. 网络绑定

- `set_network(self, net: NetSim) -> None`
  将网卡模拟器绑定到指定的网络模拟器（`NetSim` 子类，如 `SwitchNet`、`NS3BridgeNet`），同时会将网卡添加到网络模拟器的 `nics` 列表中，建立双向关联。

2. 命令参数生成

用于构建网卡模拟器的启动命令，整合各类配置参数：

- `basic_args(self, env: ExpEnv, extra: tp.Optional[str] = None) -> str`
  生成基础命令参数，包括：
  - PCIe 通信套接字路径、以太网通信套接字路径、共享内存路径
  - 同步模式（`sync_mode`）、启动时间戳（`start_tick`）、同步周期（`sync_period`）
  - PCIe 延迟（`pci_latency`）、以太网延迟（`eth_latency`）
  - 若设置了 `mac`，则附加 MAC 地址（格式转换为反向拼接字符串）
    若有额外参数（`extra`），会追加到末尾。
- `basic_run_cmd(self, env: ExpEnv, name: str, extra: tp.Optional[str] = None) -> str`
  生成完整的启动命令，格式为：`{模拟器可执行文件路径} {basic_args生成的参数}`，其中 `name` 为具体网卡模拟器的可执行文件名（如 `i40e`、`corundum-verilator`）。

3. 名称与标识

- `full_name(self) -> str`
  返回网卡模拟器的完整名称，格式为 `nic.{name}`（例如 `nic.nic0`），用于区分不同类型的模拟器。
- `is_nic(self) -> bool`
  重写自 `PCIDevSim`，返回 `True`，明确标识当前设备为网卡。

4. 套接字管理

重写 `PCIDevSim` 的方法，增加以太网通信相关的套接字管理：

- `sockets_cleanup(self, env: ExpEnv) -> tp.List[str]`
  返回实验结束后需要清理的套接字路径列表，包括：
  - 继承自 `PCIDevSim` 的 PCIe 通信套接字和共享内存路径
  - 新增的以太网通信套接字（`env.nic_eth_path(self)`）。
- `sockets_wait(self, env: ExpEnv) -> tp.List[str]`
  返回启动时需要等待的套接字路径列表，包括：
  - 继承自 `PCIDevSim` 的 PCIe 通信套接字
  - 新增的以太网通信套接字（确保网卡与网络的通信通道就绪）。



### 4. NetSim 类

`NetSim` 类是 SimBricks 框架中所有网络模拟器的基类，定义了网络模拟器的通用属性和行为，用于连接网卡模拟器（`NICSim`）及其他网络组件，实现网络层的仿真逻辑。

#### **核心属性**

| 属性名         | 类型 / 默认值              | 说明                                                         |
| -------------- | -------------------------- | ------------------------------------------------------------ |
| `opt`          | `str`（空）                | 网络模拟器的额外启动参数，用于传递自定义配置。               |
| `sync_mode`    | `int`（0）                 | 同步模式：`0` 表示非同步运行，`1` 表示同步运行（与关联组件交换同步消息）。 |
| `sync_period`  | `int`（500）               | 同步周期（纳秒），同步模式下发送同步消息的时间间隔。         |
| `eth_latency`  | `int`（500）               | 以太网延迟（纳秒），模拟网络与连接的网卡 / 主机之间的数据传输延迟。 |
| `nics`         | `List[NICSim]`             | 连接到当前网络的网卡模拟器列表，通过 `connect_nic()` 方法添加。 |
| `hosts_direct` | `List[HostSim]`            | 直接连接到当前网络的主机模拟器列表（不通过网卡，较少见）。   |
| `net_listen`   | `List[Tuple[NetSim, str]]` | 监听其他网络的连接列表（用于网络间互联），格式为 `(目标网络, 后缀)`。 |
| `net_connect`  | `List[Tuple[NetSim, str]]` | 主动连接到其他网络的列表，格式为 `(目标网络, 后缀)`。        |
| `wait`         | `bool`（False）            | 是否等待当前网络模拟器结束后再终止整个实验（通常用于控制实验生命周期）。 |

#### **核心方法**

1. 网络连接管理

- `connect_nic(self, nic: NICSim) -> None`
  将网卡模拟器（`NICSim`）连接到当前网络，同时将网卡添加到 `nics` 列表，建立网络与网卡的双向关联。
- `connect_network(self, net: NetSim, suffix: str = '') -> None`
  实现两个网络模拟器之间的互联：
  - 当前网络会主动连接到目标网络 `net`，并将连接信息添加到 `net_connect` 列表。
  - 目标网络 `net` 会监听当前网络的连接，将信息添加到 `net_listen` 列表。
  - `suffix` 用于区分同一对网络间的多个连接（如多链路场景）。

2. 套接字管理

网络模拟器通过套接字与网卡、其他网络进行通信，以下方法用于管理这些套接字路径：

- `connect_sockets(self, env: ExpEnv) -> List[Tuple[Simulator, str]]`
  返回当前网络需要主动连接的套接字列表，包括：
  - 与连接的网卡之间的以太网套接字（`env.nic_eth_path(nic)`）。
  - 与其他网络之间的互联套接字（`env.n2n_eth_path(目标网络, 当前网络, suffix)`）。
  - 与直接连接的主机之间的套接字（`env.net2host_eth_path(当前网络, 主机)`）。
- `listen_sockets(self, env: ExpEnv) -> List[Tuple[NetSim, str]]`
  返回当前网络需要监听的套接字列表，对应 `net_listen` 中记录的其他网络的连接请求（`env.n2n_eth_path(当前网络, 目标网络, suffix)`）。

3. 依赖与生命周期

- `dependencies(self) -> List[Simulator]`
  返回当前网络模拟器的依赖列表，包括：
  - 所有连接的网卡（`nics`）、直接连接的主机（`hosts_direct`）。
  - 所有需要主动连接的目标网络（`net_connect` 中的网络）。
    确保依赖的组件先启动，避免连接失败。
- `sockets_cleanup(self, env: ExpEnv) -> List[str]`
  返回实验结束后需要清理的套接字路径，即 `listen_sockets` 中监听的套接字（避免残留文件）。
- `sockets_wait(self, env: ExpEnv) -> List[str]`
  返回启动时需要等待的套接字路径，即 `listen_sockets` 中监听的套接字（确保连接就绪后再启动）。
- `wait_terminate(self) -> bool`
  返回 `wait` 属性值，控制实验是否等待当前网络模拟器终止后再结束。

4. 初始化与标识

- `full_name(self) -> str`
  返回网络模拟器的完整名称，格式为 `net.{name}`（例如 `net.switch0`），用于区分不同类型的模拟器。
- `init_network(self) -> None`
  网络初始化钩子方法，子类可重写以实现特定网络（如交换机、网桥）的初始化逻辑。



### 5. HostSim 类

`HostSim` 类是 SimBricks 框架中所有主机模拟器的基类，定义了主机模拟器的通用通用属性和行为，用于模拟计算主机（如服务器、客户端），并与网卡（`NICSim`）、PCIe 设备（`PCIDevSim`）、网络（`NetSim`）等组件交互。

#### **核心属性**

| 属性名        | 类型 / 默认值     | 说明                                                         |
| ------------- | ----------------- | ------------------------------------------------------------ |
| `node_config` | `NodeConfig`      | 主机的系统配置（如磁盘镜像、内存大小、核心数、应用程序配置等）。 |
| `wait`        | `bool`（`False`） | 是否等待该主机模拟器执行完成。`True` 表示等待其结束，`False` 表示随其他组件终止而终止。 |
| `sleep`       | `int`（0）        | 启动前的延迟时间（秒），用于控制启动顺序。                   |
| `cpu_freq`    | `str`（`4GHz`）   | 模拟的 CPU 频率（如 `5GHz`、`2GHz`）。                       |
| `sync_mode`   | `int`（0）        | 同步模式：`0` 表示非同步，`1` 表示同步（与连接的设备 / 网络交换同步消息）。 |
| `sync_period` | `int`（500）      | 同步周期（纳秒），同步模式下发送同步消息的时间间隔。         |
| `pci_latency` | `int`（500）      | PCIe 通信延迟（纳秒），模拟主机与 PCIe 设备（如网卡）的数据传输延迟。 |
| `mem_latency` | `int`（500）      | 内存 / 网络直连延迟（纳秒），模拟主机与内存设备或直接连接网络的延迟。 |
| `pcidevs`     | `List[PCIDevSim]` | 连接到主机的 PCIe 设备列表（如网卡、其他 PCI 设备）。        |
| `net_directs` | `List[NetSim]`    | 直接连接到主机的网络模拟器列表（不通过网卡，较少见）。       |
| `memdevs`     | `List[MemDevSim]` | 连接到主机的内存设备模拟器列表。                             |

#### **核心方法**

1. 设备与网络管理

- `nics(self) -> List[NICSim]`
  从 `pcidevs` 中筛选出所有网卡设备（`NICSim` 实例），方便快速获取主机上的网卡列表。
- `add_nic(self, dev: NICSim) -> None`
  为 host 添加网卡设备，内部调用 `add_pcidev` 方法，将网卡归类到 `pcidevs` 列表中。
- `add_pcidev(self, dev: PCIDevSim) -> None`
  为 host 添加 PCIe 设备（如网卡、其他 PCI 设备），同时修改设备名称（格式为 `{host.name}.{dev.name}`），建立主机与设备的关联。
- `add_memdev(self, dev: MemDevSim) -> None`
  为 host 添加内存设备，修改设备名称（格式同上），并添加到 `memdevs` 列表。
- `add_netdirect(self, net: NetSim) -> None`
  建立主机与网络的直接连接（不通过网卡），将网络添加到 `net_directs` 列表，并将主机注册到网络的 `hosts_direct` 中。

2. 依赖与生命周期

- `dependencies(self) -> List[Simulator]`
  返回主机模拟器的依赖列表，确保依赖的组件先启动：
  - 所有 PCIe 设备（`pcidevs`）及其关联的网络（如网卡连接的 `NetSim`）。
  - 所有内存设备（`memdevs`）。
- `wait_terminate(self) -> bool`
  返回 `wait` 属性值，控制实验是否等待当前主机模拟器终止后再结束。

3. 标识与资源

- `full_name(self) -> str`
  返回主机模拟器的完整名称，格式为 `host.{name}`（例如 `host.server.0`），用于区分不同类型的模拟器。
- 继承自 `Simulator` 的资源方法：
  - `resreq_cores(self) -> int`：子类实现，返回所需 CPU 核心数。
  - `resreq_mem(self) -> int`：子类实现，返回所需内存大小（MB）。

#### **典型子类**

`HostSim` 是抽象基类，具体主机模拟器通过继承它实现不同的仿真逻辑：

- **`QemuHost`**：基于 QEMU 的主机模拟器，支持 KVM 加速，适用于快速仿真。
  - 关键属性：`sync`（是否与设备同步）、`cpu_freq`（CPU 频率）。
  - 核心方法：`run_cmd` 生成 QEMU 启动命令，包含磁盘、内核、PCI 设备等配置。
- **`Gem5Host`**：基于 gem5 的主机模拟器，支持精确的 CPU 时序仿真（如 `TimingSimpleCPU`、`DerivO3CPU`）。
  - 关键属性：`cpu_type`（CPU 模型）、`sys_clock`（系统时钟）、`variant`（gem5 编译变体）。
  - 核心方法：`run_cmd` 生成 gem5 启动命令，配置缓存、内存、PCIe 设备等参数，支持 checkpoint 功能。
- **`SimicsHost`**：基于 Simics 的主机模拟器，适用于高精度硬件仿真。
  - 关键属性：`cpu_class`（CPU 类型）、`timing`（是否启用时序模型）、`interactive`（是否交互模式）。



### 6. QemuHost 类

主要进行启动指令的分析，其余内容与父类 HostSim 类似。

1. **`prep_cmds` 方法：预处理命令**

```python
def prep_cmds(self, env: ExpEnv) -> tp.List[str]:
    return [
        f'{env.qemu_img_path} create -f qcow2 -o '
        f'backing_file="{env.hd_path(self.node_config.disk_image)}" '
        f'{env.hdcopy_path(self)}'
    ]
```

- **功能**：创建一个基于原始磁盘镜像的可写副本（COW 机制），避免直接修改原始镜像。
- **命令拆解**：
  - `{env.qemu_img_path}`：QEMU 镜像工具路径（如 `qemu-img`）。
  - `create -f qcow2`：创建 QCOW2 格式的镜像（支持 Copy-On-Write，节省空间）。
  - `-o backing_file="..."`：指定原始镜像为后端（只读），新镜像仅存储修改内容。
  - `{env.hdcopy_path(self)}`：新镜像的保存路径（用于当前主机模拟器的磁盘）。

2. **`run_cmd` 方法：QEMU 启动命令**

生成完整的 QEMU 启动指令，包含硬件配置、设备挂载、同步参数等，核心逻辑如下：

2.1 基础配置

```python
accel = ',accel=kvm:tcg' if not self.sync else ''  # 加速模式：非同步时启用 KVM/TCG
kcmd_append = ' ' + self.node_config.kcmd_append if self.node_config.kcmd_append else ''  # 内核命令行附加参数
```

2.2 核心命令框架

```bash
{env.qemu_path} -machine q35{accel} -serial mon:stdio \
  -cpu Skylake-Server -display none -nic none \
  -kernel {env.qemu_kernel_path} \
  -drive file={env.hdcopy_path(self)},if=ide,index=0,media=disk \
  -drive file={env.cfgtar_path(self)},if=ide,index=1,media=disk,driver=raw \
  -append "earlyprintk=ttyS0 console=ttyS0 root=/dev/sda1 init=/home/ubuntu/guestinit.sh rw{kcmd_append}" \
  -m {self.node_config.memory} -smp {self.node_config.cores}
```

- **关键参数解析**：
  - `{env.qemu_path}`：QEMU 主程序路径（如 `qemu-system-x86_64`）。
  - `-machine q35{accel}`：使用 Q35 芯片组，非同步模式下附加 `accel=kvm:tcg`（优先 KVM 硬件加速， fallback 到 TCG 软件模拟）。
  - `-serial mon:stdio`：将串口输出重定向到控制台，便于调试。
  - `-cpu Skylake-Server`：模拟 Intel Skylake 服务器 CPU。
  - `-display none`：禁用图形界面（无头模式）。
  - `-nic none`：禁用默认网卡（由 SimBricks 自定义设备接管）。
  - `-kernel {env.qemu_kernel_path}`：指定启动内核镜像路径。
  - `-drive ...`：挂载磁盘：
    - 第一个 drive：挂载预处理生成的 QCOW2 镜像（主磁盘，`/dev/sda1`）。
    - 第二个 drive：挂载配置文件 tar 包（只读，用于传递实验配置）。
  - `-append "..."`：内核启动参数：
    - `console=ttyS0`：串口作为控制台。
    - `root=/dev/sda1`：根文件系统挂载点。
    - `init=/home/ubuntu/guestinit.sh`：自定义初始化脚本（启动实验应用）。
    - `rw`：根文件系统可写。
  - `-m {memory}`：分配的内存大小（来自 `node_config`）。
  - `-smp {cores}`：模拟的 CPU 核心数（来自 `node_config`）。

2.3 同步模式配置（`self.sync = True` 时）

```python
# 计算 CPU 频率对应的时间偏移（用于同步时序）
unit = self.cpu_freq[-3:]  # 提取单位（GHz 或 MHz）
base = 0 if unit.lower() == 'ghz' else 3  # GHz 对应 base=0，MHz 对应 base=3
num = float(self.cpu_freq[:-3])  # 提取频率数值（如 4GHz -> 4.0）
shift = base - int(math.ceil(math.log(num, 2)))  # 计算偏移量

cmd += f' -icount shift={shift},sleep=off '  # 启用指令计数和时序同步
```

- **作用**：当启用同步模式时，通过 `-icount` 参数控制 QEMU 的指令执行速度，确保与其他模拟器（如网卡、网络）的时序一致性。
  - `shift`：调整指令计数的时间缩放因子，使 QEMU 模拟速度匹配实际时间。
  - `sleep=off`：禁用 QEMU 的休眠优化，避免同步漂移。

2.4 挂载 PCIe 设备（如网卡）

```python
for dev in self.pcidevs:
    cmd += f'-device simbricks-pci,socket={env.dev_pci_path(dev)}'
    if self.sync:
        cmd += ',sync=on'
        cmd += f',pci-latency={self.pci_latency}'
        cmd += f',sync-period={self.sync_period}'
    else:
        cmd += ',sync=off'
    cmd += ' '
```

- **功能**：为 QEMU 添加 SimBricks 自定义 PCIe 设备（如网卡）。
- **参数解析**：
  - `simbricks-pci`：SimBricks 实现的 PCIe 设备驱动。
  - `socket={...}`：指定与设备通信的 Unix 套接字路径。
  - 同步模式下：
    - `sync=on`：启用设备同步。
    - `pci-latency`：PCIe 通信延迟（纳秒）。
    - `sync-period`：同步消息发送周期（纳秒）。



### 7. Gem5Host 类

用于启动 gem5 模拟器，分析其属性与启动指令。

#### **核心属性**

| 属性名                   | 类型 / 默认值              | 说明                                                         |
| ------------------------ | -------------------------- | ------------------------------------------------------------ |
| `cpu_type_cp`            | `str`（`X86KvmCPU`）       | 生成检查点（checkpoint）时使用的 CPU 模型（KVM 加速，快速启动）。 |
| `cpu_type`               | `str`（`TimingSimpleCPU`） | 实际仿真时使用的 CPU 模型（时序精确模型，支持详细的指令级仿真）。 |
| `sys_clock`              | `str`（`1GHz`）            | 系统时钟频率（如 `2GHz`）。                                  |
| `variant`                | `str`（`fast`）            | gem5 编译变体（如 `fast` 为优化版本，`debug` 为调试版本）。  |
| `modify_checkpoint_tick` | `bool`（`True`）           | 恢复检查点时是否将事件队列 tick 重置为 0（优化同步性能）。   |
| `extra_main_args`        | `List[str]`                | 传递给 gem5 主程序的额外参数。                               |
| `extra_config_args`      | `List[str]`                | 传递给 gem5 配置脚本（Python）的额外参数。                   |

#### **核心方法**

1. **资源需求**

- `resreq_cores(self) -> int`
  返回所需 CPU 核心数，固定为 1（gem5 本身通常单进程运行，多核心通过内部模拟实现）。
- `resreq_mem(self) -> int`
  返回所需内存大小，固定为 4096 MB（4GB，满足 gem5 高精度仿真的内存需求）。

2. **预处理命令（`prep_cmds`）**

```python
def prep_cmds(self, env: ExpEnv) -> tp.List[str]:
    cmds = [f'mkdir -p {env.gem5_cpdir(self)}']  # 创建检查点目录
    if env.restore_cp and self.modify_checkpoint_tick: # (此处代码 Docker image 与仓库中有点不同)
        # 恢复检查点前，将事件队列 tick 重置为 0（优化同步）
        cmds.append(
            f'python3 {env.utilsdir}/modify_gem5_cp_tick.py --tick 0 '
            f'--cpdir {env.gem5_cpdir(self)}'
        )
    return cmds
```

- **功能**：准备 gem5 运行环境，包括创建检查点目录和调整检查点的时间戳（优化多模拟器同步）。

3. **启动命令（`run_cmd`）**

生成 gem5 完整启动指令，包含硬件配置、检查点控制、设备连接等参数，核心逻辑如下：

3.1 基础配置

```python
# 根据是否生成检查点选择 CPU 模型（检查点阶段用 KVM 加速，仿真阶段用时序模型）
cpu_type = self.cpu_type if not env.create_cp else self.cpu_type_cp

cmd = f'{env.gem5_path(self.variant)} --outdir={env.gem5_outdir(self)} '
cmd += ' '.join(self.extra_main_args)  # 附加主程序参数
```

3.2 系统与硬件配置

```bash
{env.gem5_path} --outdir={输出目录} \
  {extra_main_args} \
  {env.gem5_py_path} --caches --l2cache \
  --l1d_size=32kB --l1i_size=32kB --l2_size=32MB \
  --l1d_assoc=8 --l1i_assoc=8 --l2_assoc=16 \
  --cacheline_size=64 --cpu-clock={self.cpu_freq} \
  --sys-clock={self.sys_clock} \
  --checkpoint-dir={检查点目录} \
  --kernel={内核镜像路径} \
  --disk-image={磁盘镜像路径} \
  --disk-image={配置文件tar包路径} \
  --cpu-type={cpu_type} --mem-size={内存大小}MB \
  --num-cpus={核心数} \
  --mem-type=DDR4_2400_16x4
```

- **关键参数解析**：
  - `--caches --l2cache`：启用 L1（指令 / 数据）和 L2 缓存模拟。
  - 缓存参数（`l1d_size`、`l2_assoc` 等）：定义缓存大小、相联度、行大小等细节。
  - `--cpu-clock`/`--sys-clock`：CPU 和系统时钟频率。
  - `--kernel`/`--disk-image`：指定内核镜像和磁盘镜像（主磁盘 + 配置文件 tar 包）。
  - `--cpu-type`：CPU 模型（`TimingSimpleCPU` 为时序精确模型，`X86KvmCPU` 为 KVM 加速模型）。
  - `--mem-type`：内存类型（如 DDR4 2400MHz）。

3.3 检查点控制

```python
if env.create_cp:
    cmd += '--max-checkpoints=1 '  # 生成检查点（最多1个）
if env.restore_cp:
    cmd += '-r 1 '  # 从检查点恢复（恢复第1个检查点）
```

- **功能**：控制 gem5 的检查点生成与恢复，用于加速仿真（跳过重复的启动过程）。

3.4 设备连接（PCIe 设备、内存设备、直接网络）

- **PCIe 设备（如网卡）**：

  ```python
  for dev in self.pcidevs:
      cmd += (
          f'--simbricks-pci=connect:{env.dev_pci_path(dev)}'
          f':latency={self.pci_latency}ns'
          f':sync_interval={self.sync_period}ns'
      )
      if cpu_type == 'TimingSimpleCPU':  # 时序模型下启用同步
          cmd += ':sync'
      cmd += ' '
  ```

  - 通过 `--simbricks-pci` 连接自定义 PCIe 设备，指定通信套接字路径、延迟和同步周期。

- **内存设备**：

  ```python
  for dev in self.memdevs:
      cmd += (
          f'--simbricks-mem={dev.size}@{dev.addr}@{dev.as_id}@'
          f'connect:{env.dev_mem_path(dev)}'
          f':latency={self.mem_latency}ns'
          f':sync_interval={self.sync_period}ns'
      )
      if cpu_type == 'TimingSimpleCPU':
          cmd += ':sync'
      cmd += ' '
  ```

  - 通过 `--simbricks-mem` 连接内存设备，指定内存大小、地址、通信路径和延迟。

- **直接网络连接**：

  ```python
  for net in self.net_directs:
      cmd += (
          '--simbricks-eth-e1000=listen'
          f':{env.net2host_eth_path(net, self)}'
          f':{env.net2host_shm_path(net, self)}'
          f':latency={net.eth_latency}ns'
          f':sync_interval={net.sync_period}ns'
      )
      if cpu_type == 'TimingSimpleCPU':
          cmd += ':sync'
      cmd += ' '
  ```

  - 通过 `--simbricks-eth-e1000` 直接连接网络（不通过网卡），模拟主机与网络的直连通信。





剩余模拟器类不再过多赘述，不同网卡模拟器基本上只是启动不同的程序