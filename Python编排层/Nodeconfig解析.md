# nodeconfig.py

---

### 1. 节点定义（主机）

**NodeConfig** 是 Simbricks仿真框架中用于定义节点或主机配置的核心类，提供了统一的接口来管理节点的各种参数和行为，使得仿真实验的配置更加灵活和可扩展。

1. **基本配置属性**：

   - 定义仿真节点的基础参数，如IP地址(`ip`)、子网前缀(`prefix`)、CPU核心数(`cores`)、内存大小(`memory`)等
   - 指定使用的磁盘镜像(`disk_image`)和网络MTU大小(`mtu`)
   - 配置TCP拥塞控制算法(`tcp_congestion_control`)

2. **生命周期管理**：

   - 提供准备阶段命令(`prepare_pre_cp`和`prepare_post_cp`)
   - 定义运行命令(`run_cmds`)和清理命令(`cleanup_cmds`)
   - 支持创建包含配置文件的tar包(`make_tar`)

3. **应用集成**：

   - 通过`AppConfig`类关联要在节点上运行的应用程序
   - 支持通过`config_files`方法添加额外的配置文件

4. **派生类支持**：

   - 提供了多种特定类型的节点配置，如`LinuxNode`、`MtcpNode`、`TASNode`等
   - 每个派生类可以覆盖基础方法来实现特定行为

5. **仿真器适配**：

   - 通过`sim`属性指定使用的仿真器(qemu/gem5等)
   - 根据仿真器类型生成不同的控制命令(如checkpoint和exit命令)

   

在 **LinuxNode**类的初始化方法中，首先调用了父类 `NodeConfig` 的初始化方法。然后，定义了以下四个属性：

- `ifname`: 网络接口的名称，默认值为 `'eth0'`。
- `drivers`: 一个字符串列表，用于存储需要加载的内核模块或驱动程序的名称。
- `force_mac_addr`: 一个可选的字符串，用于指定网络接口的 MAC 地址。
- `hostname`: 一个可选的字符串，用于指定节点的主机名，默认值为 `'ubuntu'`。

`LinuxNode` 类重写了父类 `NodeConfig` 中的两个方法：`prepare_pre_cp` 和 `prepare_post_cp`。

prepare_pre_cp 用于准备在节点进行检查点之前需要执行的命令。它首先调用父类的 `prepare_pre_cp` 方法获取一组命令，然后根据 `hostname` 属性，如果设置了主机名，则添加设置主机名的命令。

prepare_post_cp 用于准备在节点保存检查点之后需要执行的命令。它首先根据 `drivers` 属性加载内核模块或驱动程序，然后如果设置了 `force_mac_addr`，则设置网络接口的 MAC 地址，并启用网络接口，最后添加 IP 地址。最后，调用父类的 `prepare_post_cp` 方法获取并添加其他命令。



- **I40eLinuxNode** 只是在`LinuxNode`基础上，在驱动里加了一个 **i40e**

- **CorundumLinuxNode** 加了一个 **/tmp/guest/mqnic.ko**

- **E1000LinuxNode** 加了一个 **e1000**



### 2. APP 定义（应用）

`AppConfig` 类用于描述在节点或主机上运行的应用程序。这个类提供了一系列方法，用于配置和管理应用程序的执行。

`AppConfig` 类通常与 `NodeConfig` 类一起使用，`NodeConfig` 类定义了节点的配置。在实验中，可以创建不同的 `AppConfig` 子类来配置不同类型的应用程序，并将它们分配给 `NodeConfig` 实例，以便在模拟节点上运行。

1. **PingClient**

`PingClient` 类通过提供一个简单的网络测试工具，在模拟的网络环境中评估节点之间的连通性。通过配置不同的 `server_ip`，可以测试不同节点之间的网络连接情况。

- `server_ip`: 这是一个可选参数，默认值为 `'192.168.64.1'`。

- 向指定的 `server_ip` 发送 10 个 ICMP ECHO 请求包。

- 命令格式是 `f'ping {self.server_ip} -c 10'`

2. **IperfTCPServer**

IperfTCPServer 是一个基于 Iperf 工具的网络性能测试服务器。Iperf 是一款开源的网络带宽测试工具，用于测量 TCP 和 UDP 的最大带宽性能。命令 `iperf -s -l 32M -w 32M` 启动了一个 Iperf TCP 服务器，配置了 32MB 的数据包大小和 TCP 窗口大小。

3. **IperfUDPServer**

启动一个 iperf 服务端，监听 UDP 连接，以便客户端可以连接到这个服务端进行网络带宽和延迟测试。

4. **IperfTCPClient**

定义一系列命令来配置和启动一个 iperf TCP 客户端，这些命令将在模拟网络环境中的节点上执行。通过调整 `server_ip`、`procs` 和 `is_last` 属性，可以定制客户端的行为以适应不同的测试场景。

5. **IperfUDPClient**

通过配置 `iperf` 工具，允许在模拟的网络环境中进行 UDP 性能测试。通过调整 `server_ip` 和 `rate` 属性，可以指定测试的目标服务器和数据流速率。

---

### 1. 网络性能基准测试 (Network Benchmarking) 🌐



这是最主要的一类，用于全面衡量模拟网络的性能。

- **`iperf`**: 这是支持最丰富的工具，提供了大量的客户端和服务器类。
  - **标准 TCP/UDP**: 由 `IperfTCPServer`/`Client` 和 `IperfUDPServer`/`Client` 支持，用于测试基础的吞吐量。
  - **DCTCP**: 由 `DctcpServer` 和 `DctcpClient` 支持，专门用于测试数据中心 TCP 拥塞控制算法。
  - **通用 TCP 拥塞控制**: 由 `TcpCongServer` 和 `TcpCongClient` 支持，可以用来测试不同的 TCP 算法。
- **`netperf`**: 由 `NetperfServer` 和 `NetperfClient` 类支持。是另一个强大的网络测试工具，除了测试吞吐量，还能精确测量请求/响应（RR）延迟。
- **`Ping`**: 由 `PingClient` 类支持，用于测试最基本的网络连通性和往返时延 (RTT)。

------



### 2. Web 服务器与客户端 (Web Server & Client) 🖥️



用于模拟 HTTP Web 流量。

- **`lighttpd` 服务器**: 由 `HTTPD` 及其子类 (`HTTPDLinux`, `HTTPDMtcp` 等) 支持。它可以在模拟的服务器上运行一个轻量级的 `lighttpd` Web 服务。
- **Apache Bench (`ab`) 客户端**: 由 `HTTPC` 及其子类 (`HTTPCLinux`, `HTTPCMtcp`) 支持。它使用 `ab` 工具向服务器发起并发请求，以测试 Web 服务器的性能。

------



### 3. 分布式共识协议 (Distributed Consensus) 🤝



这部分用于前沿的分布式系统研究，模拟确保系统一致性的算法。

- **`NOPaxos`**: 提供了完整的组件支持，包括 `NOPaxosReplica` (副本)、`NOPaxosClient` (客户端) 和 `NOPaxosSequencer` (序列器)。
- **`Viewstamped Replication (VR)`**: 由 `VRReplica` 和 `VRClient` 支持，是另一种经典的共识算法。

------



### 4. 键值存储 / 缓存系统 (Key-Value Stores / Caching) 🗄️



用于模拟 `Memcached` 这样的分布式缓存系统。

- **`Memcached`**: 由 `MemcachedServer` 和 `MemcachedClient` 支持。服务器运行 `memcached` 服务，客户端则使用 `memaslap` 工具来产生高并发的负载进行压测。

------



### 5. 远程过程调用 (RPC) 微基准测试 🔄



- **`RPC`**: 由 `RPCServer` 和 `RPCClient` 支持。这是一个来自 `tasbench` 项目的微基准测试，用于模拟大量短小的远程过程调用，以测试网络栈的处理能力。

