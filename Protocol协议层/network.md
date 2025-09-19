### 2. 网络协议（network/proto.h、network/if.h、network/if.c）

- **核心作用**：定义网络设备间的通信协议，主要用于传输网络数据包。
- **关键结构与宏**：
  - **初始化消息**：`SimbricksProtoNetIntro` 是网络设备间的握手消息（目前仅含占位字段）。
  - **数据包消息**：`SimbricksProtoNetMsgPacket` 用于传输网络包，包含包长度（`len`）、端口（`port`）、时间戳（`timestamp`）及数据（`data[]`）。
  - **消息类型**：`SIMBRICKS_PROTO_NET_MSG_PACKET` 标识网络包消息。
  - **接口初始化**：`SimbricksNetIfInit` 函数负责初始化网络接口，建立连接并完成握手。