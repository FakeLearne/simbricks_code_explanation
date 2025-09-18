代码位置 experiment/simbricks/orchestration/experiment/*



### ExpEnv 类

`ExpEnv` 类是 SimBricks 框架中用于管理实验环境的核心类，负责维护仿真实验所需的路径、配置文件、工具路径等环境信息，为各类各类模拟器（如 gem5、QEMU、Simics 等）提供统一的路径管理和资源访问接口。

#### 1. **核心属性**

| 属性名                                         | 说明                                                         |
| ---------------------------------------------- | ------------------------------------------------------------ |
| `create_cp`                                    | 布尔值，标识是否需要生成检查点（checkpoint）。               |
| `restore_cp`                                   | 布尔值，标识是否需要从检查点恢复仿真状态。                   |
| `pcap_file`                                    | PCAP 包捕获文件路径（若启用网络包捕获）。                    |
| `repodir`                                      | SimBricks 代码仓库的绝对路径（通过初始化参数传入）。         |
| `workdir`                                      | 工作目录的绝对路径，用于存放临时文件、日志、套接字等（通过参数传入）。 |
| `cpdir`                                        | 检查点目录的绝对路径，用于存储和读取检查点文件（通过参数传入）。 |
| `shm_base`                                     | 共享内存目录的基础路径，默认与 `workdir` 一致。              |
| `utilsdir`                                     | 工具脚本 / 二进制文件的目录路径（位于仓库内的 `experiments/simbricks/utils`）。 |
| 各类工具路径（如 `qemu_path`、`gem5_path` 等） | 预定义的模拟器和辅助工具的路径（基于 `repodir` 计算）。      |

#### 2. **核心方法**

1. 工具路径获取

- `gem5_path(self, variant: str) -> str`
  根据编译变体（如 `opt`、`debug`）返回 gem5 可执行文件的路径，例如：
  `{repodir}/sims/external/gem5/build/X86/gem5.fast`。
- 其他工具路径（如 `qemu_path`、`simics_path` 等）
  预定义了 QEMU、Simics 等模拟器的可执行文件路径，以及内核镜像（`qemu_kernel_path`、`gem5_kernel_path`）的位置，避免硬编码路径导致的维护问题。

2. 路径生成与管理

这些方法为不同类型的模拟器、设备、网络组件生成统一格式的路径，确保仿真组件之间的通信路径一致。

| 方法名                            | 生成路径示例                        | 用途                                                         |
| --------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| `hd_path(hd_name_or_path)`        | `{repodir}/images/output-hd1/hd1`   | 返回磁盘镜像文件路径（支持绝对路径直接使用，相对路径自动拼接仓库镜像目录）。 |
| `dev_pci_path(sim)`               | `{workdir}/dev.pci.nic0`            | 生成 PCIe 设备（如网卡）的套接字路径，用于主机与设备的通信。 |
| `nic_eth_path(sim)`               | `{workdir}/nic.eth.nic0`            | 生成网卡与网络模拟器之间的以太网通信套接字路径。             |
| `n2n_eth_path(sim_l, sim_c)`      | `{workdir}/n2n.eth.switch0.router0` | 生成两个网络组件（如交换机与路由器）之间的通信套接字路径。   |
| `net2host_eth_path(sim_n, sim_h)` | `{workdir}/n2h.eth.net0.host0`      | 生成网络模拟器与主机之间的直接通信套接字路径。               |
| `gem5_outdir(sim)`                | `{workdir}/gem5-out.host0`          | 生成 gem5 模拟器的输出目录（存放日志、统计信息等）。         |
| `gem5_cpdir(sim)`                 | `{cpdir}/gem5-cp.host0`             | 生成 gem5 检查点的存储目录。                                 |

3. 辅助功能

- `is_absolute_exists(path: str) -> bool`
  检查路径是否为绝对路径且文件存在，用于区分用户传入的绝对路径和相对路径（相对路径会自动拼接仓库内的默认目录）。





### IPC in Simbricks

在 Simbricks 框架中，不同模拟器进程间的通信方式主要通过**Unix 域套接字（Unix Domain Sockets）** 和**共享内存（Shared Memory）** 实现，具体取决于组件类型和交互场景，可从代码中总结如下：

#### 1. **Unix 域套接字（主要通信方式）**

Unix 域套接字是模拟器进程间最核心的通信方式，适用于同一主机内不同组件（如主机模拟器、NIC 模拟器、网络模拟器）的交互，通过文件系统路径标识通信端点。

- **主机与 NIC 通信（PCIe 交互）**
  主机模拟器（如 QemuHost、Gem5Host、SimicsHost）与 NIC 模拟器（如 I40eNIC、CorundumBMNIC）通过 PCIe 虚拟设备通信，使用 `dev_pci_path` 生成套接字路径：

  ```python
  # 示例路径：workdir/dev.pci.nic0
  def dev_pci_path(self, sim) -> str:
      return f'{self.workdir}/dev.pci.{sim.name}'
  ```

  例如，Gem5Host 的 `run_cmd` 中会通过该路径创建 PCIe 设备通信套接字，实现主机与 NIC 的命令和数据交互。

- **NIC 与网络模拟器通信（以太网交互）**
  NIC 模拟器与网络模拟器（如 SwitchNet、NS3BridgeNet）通过以太网虚拟链路通信，使用 `nic_eth_path` 或 `n2n_eth_path` 生成套接字路径：

  ```python
  # NIC 与网络的以太网套接字：workdir/nic.eth.nic0
  def nic_eth_path(self, sim: 'simulators.Simulator') -> str:
      return f'{self.workdir}/nic.eth.{sim.name}'
  
  # 网络组件间的以太网套接字：workdir/n2n.eth.switch0.router0
  def n2n_eth_path(self, sim_l, sim_c, suffix='') -> str:
      return f'{self.workdir}/n2n.eth.{sim_l.name}.{sim_c.name}.{suffix}'
  ```

  例如，SwitchNet 与 I40eNIC 会通过此类套接字传递以太网帧。

- **网络与主机直接通信**
  部分场景下网络模拟器与主机直接交互，使用 `net2host_eth_path` 生成套接字路径：

  ```python
  # 网络到主机的套接字：workdir/n2h.eth.net0.host0
  def net2host_eth_path(self, sim_n, sim_h) -> str:
      return f'{self.workdir}/n2h.eth.{sim_n.name}.{sim_h.name}'
  ```

#### 2. **共享内存（辅助高性能通信）**

共享内存适用于需要高频数据交换的场景（如高性能 NIC 与主机的内存映射交互），通过文件系统路径标识共享内存区域。

- **设备与主机的内存映射通信**
  部分设备模拟器（如 MemNIC、FEMUDev）通过共享内存暴露内存区域，使用 `dev_shm_path` 或 `proxy_shm_path` 生成路径：

  ```python
  # 设备共享内存路径：shm_base/dev.shm.nic0
  def dev_shm_path(self, sim: 'simulators.Simulator') -> str:
      return f'{self.shm_base}/dev.shm.{sim.name}'
  
  # 代理共享内存路径：shm_base/proxy.shm.proxy0
  def proxy_shm_path(self, sim: 'simulators.Simulator') -> str:
      return f'{self.shm_base}/proxy.shm.{sim.name}'
  ```

  例如，Corundum 系列 NIC 可能通过共享内存实现高效的数据包 DMA 传输。

- **网络与主机的共享内存通信**
  部分网络模拟器（如 MemSwitchNet）支持将网络流量映射到主机内存，使用 `net2host_shm_path` 生成路径：

  ```python
  # 网络到主机的共享内存路径：workdir/n2h.shm.net0.host0
  def net2host_shm_path(self, sim_n, sim_h) -> str:
      return f'{self.workdir}/n2h.shm.{sim_n.name}.{sim_h.name}'
  ```