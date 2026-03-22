# 网络扫描工具

# 2425a 计算机网络课程设计
网络小工具, 包括三个功能: 端口扫描(PortScan)、 路由追踪(TraceRoute)、 DNS查询(DNSQuery)

# 功能实现:
- 端口扫描（PortScan）
     - 协议层原理：基于 TCP 主动连接扫描（Connect Scan）。
     - 实现方式：后端并发对端口区间 `[start, end]` 发起 `TCP SYN -> SYN/ACK` 建连流程（由系统 `connect`/`DialTimeout` 封装）。
     - 判定逻辑：
          - 连接成功：端口开放（Open）；
          - 连接超时或被拒绝：端口关闭或被过滤（Closed/Filtered）。
     - 特点：通过超时参数控制探测时长，结果以流式方式逐个返回开放端口。

- 路由追踪（TraceRoute）
     - 协议层原理：基于 ICMP Echo Request/Echo Reply 与 IP 头 TTL 递增机制。
     - 实现：
          - 使用原始套接字 `SOCK_RAW + IPPROTO_ICMP` 构造 ICMP 探测包；
          - 逐跳设置 TTL（`first_ttl -> max_ttl`）；
          - 中间路由器在 TTL 归零时返回 `ICMP Time Exceeded`，据此识别当前跳地址；
          - 到达目标主机后收到 `ICMP Echo Reply`，追踪结束。
     - 时延测量：每跳发送多次探测（`nqueries`），记录毫秒级 RTT，超时以 `*` 标记。

- DNS查询（DNSQuery）
     - 协议层原理：基于 DNS 应用层协议，使用 UDP/53 进行迭代解析。
     - 实现：
          - 手动构造 DNS 报文头与 Question 区（支持 `A`/`AAAA` 查询）；
          - 从根服务器开始迭代查询，按 `NS` 与附加记录中的地址逐级下钻；
          - 支持域名压缩指针解码（Name Compression）；
          - 解析 `A/AAAA/NS/CNAME` 资源记录并输出 TTL；
          - 遇到 `CNAME` 时递归查询别名目标，并带简单缓存与递归深度限制。
     - 结果形式：实时输出“查询过程节点 + 最终解析结果（IP 与 TTL）”。

- 后端接口:
     - 服务框架：Go + Gin，提供 HTTP 与 WebSocket 能力。
     - 路由：
          - `GET /ping`：健康检查；
          - `GET /ws`：统一任务入口（WebSocket）。
     - 请求模型：前端发送 `{ type, target, traceroute, portscan, dnsquery }`。
          - `type = portscan | traceroute | dnsquery` 用于分发。
     - 执行链路：
          - `portscan`：Go 并发扫描并流式回传开放端口；
          - `traceroute`：调用 `cppmodules/traceroute`，逐行解析并转 JSON 回传；
          - `dnsquery`：调用 `cppmodules/dnsquery`，逐行转发查询过程与结果。
     - 通信特征：单连接、实时推送、边计算边返回，便于前端增量渲染。

- 前端页面:
     - 技术栈：Svelte + Rollup。
     - 交互入口：主页输入目标主机（IP/域名），按校验规则启用对应功能按钮。
     - 三个功能视图：
          - `DnsQueryView`：建立 WebSocket 后展示 DNS 迭代查询路径与最终记录；
          - `TraceRouteView`：参数化发起路由追踪，按跳展示地址与多次 RTT；
          - `PortScanView`：配置端口范围与超时，实时追加开放端口列表。
     - 报告能力：支持将当前结果区域截图导出为 PNG 报告。
     - 体验特性：流式结果渲染、参数表单化、状态提示（进行中/完成）。  
  
  
### 相关技术栈
前端: svelte -rollup -node
后端: go -gin -gorilla/websocket
     c++ -posix api