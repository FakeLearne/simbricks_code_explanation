1. 创建节点（主机）

```python
# create client
client_config = I40eLinuxNode()  # boot Linux with i40e NIC driver
client_config.ip = '10.0.0.1'
client_config.app = PingClient(server_ip='10.0.0.2')
client = Gem5Host(client_config)
client.name = 'client'
client.wait = True  # wait for client simulator to finish execution
e.add_host(client)
```

I40eLinuxNode 继承自 nodeconfig

