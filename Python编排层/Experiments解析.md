代码位置 experiment/simbricks/orchestration/experiments.py

`Experiment` 类是 SimBricks 框架中所有仿真实验的基类，用于定义实验的核心属性、组件集合及管理方法，是构建和描述仿真实验的基础，描述了所有要运行的模拟器。



### **核心属性**

类通过 `__init__` 方法初始化实验的基本配置和组件容器，主要包括：

| 属性名         | 类型 / 默认值          | 说明                                                         |
| -------------- | ---------------------- | ------------------------------------------------------------ |
| `name`         | `str`                  | 实验名称，用于标识和筛选实验。                               |
| `timeout`      | `Optional[int]`        | 实验超时时间（秒），未设置则无超时限制。                     |
| `checkpoint`   | `bool`（默认 `False`） | 是否启用 checkpoint 机制（如快速启动主机模拟器，通过 checkpoint 跳过启动阶段）。 |
| `no_simbricks` | `bool`（默认 `False`） | 是否禁用 SimBricks 适配器（若为 `True`，所有模拟器不使用框架提供的适配器）。 |
| `hosts`        | `List[HostSim]`        | 存储实验中所有主机模拟器（如 QEMU 等）。                     |
| `pcidevs`      | `List[PCIDevSim]`      | 存储所有 PCIe 设备模拟器（如网卡、GPU 等）。                 |
| `memdevs`      | `List[MemDevSim]`      | 存储所有内存设备模拟器。                                     |
| `netmems`      | `List[NetMemSim]`      | 存储所有网络内存模拟器。                                     |
| `networks`     | `List[NetSim]`         | 存储所有网络模拟器（如 switch、router 等）。                 |
| `metadata`     | `Dict[str, Any]`       | 实验元数据（如自定义标签、参数等，用于记录额外信息）。       |



### **核心方法**

#### 1. 组件管理方法

用于向实验中添加各类模拟器组件，并确保组件名称唯一（避免冲突）：

- `add_host(sim: HostSim)`：添加主机模拟器，检查并禁止重复名称。
- `add_nic(sim: Union[NICSim, I40eMultiNIC])`：添加网卡模拟器（本质是调用 `add_pcidev`，因网卡属于 PCIe 设备）。
- `add_pcidev(sim: PCIDevSim)`：添加 PCIe 设备模拟器，检查名称唯一性。
- `add_memdev(sim: MemDevSim)`：添加内存设备模拟器，检查名称唯一性。
- `add_netmem(sim: NetMemSim)`：添加网络内存模拟器，检查名称唯一性。
- `add_network(sim: NetSim)`：添加网络模拟器，检查名称唯一性。

#### 2. 组件查询与聚合

- `@property nics`：通过过滤 `pcidevs` 中标记为网卡（`is_nic()` 返回 `True`）的设备，便捷获取所有网卡模拟器。
- `all_simulators() -> Iterable[Simulator]`：聚合所有类型的模拟器（主机、PCIe 设备、内存设备、网络内存、网络），返回一个可迭代对象，用于统一处理所有组件（如依赖分析、资源计算）。

#### 3. 资源需求计算

用于评估实验所需的系统资源，辅助调度和资源分配：

- `resreq_mem() -> int`：计算所有模拟器所需的总内存（累加每个模拟器的 `resreq_mem()` 结果）。
- `resreq_cores() -> int`：计算所有模拟器所需的总核心数（累加每个模拟器的 `resreq_cores()` 结果）。



### **设计意义**

- **统一管理**：通过分类容器（`hosts`、`pcidevs` 等）和添加方法，规范实验组件的管理，确保名称唯一，避免冲突。
- **灵活性**：支持多种类型的模拟器（主机、网络、设备等），适用于构建复杂的系统仿真场景（如分布式网络、多设备交互）。
- **资源评估**：通过 `resreq_mem` 和 `resreq_cores` 提前评估资源需求，帮助框架判断实验是否可在当前环境运行。



### **使用场景**

用户定义实验时，通常会实例化 `Experiment` 类，通过 `add_*` 方法添加所需的模拟器（如主机、网卡、网络），设置超时、checkpoint 等参数，最终由框架的运行时（Runtime）和执行器（Runner）加载并执行。例如：

```python
from simbricks.orchestration.experiments import Experiment
from simbricks.orchestration.simulators import HostSim, NetSim, NICSim

# 创建实验
exp = Experiment(name="my_exp")
exp.timeout = 3600  # 1小时超时

# 添加组件
host = HostSim(name="host1")
nic = NICSim(name="nic1")
net = NetSim(name="net1")

exp.add_host(host)
exp.add_nic(nic)
exp.add_network(net)

# 查看总资源需求
print(f"所需内存: {exp.resreq_mem()} MB")
print(f"所需核心: {exp.resreq_cores()}")
```

该类是连接用户实验定义与框架执行逻辑的关键，封装了实验的核心元信息和组件集合。