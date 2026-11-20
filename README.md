# shred-decode

基于 Rust 的高性能 Solana **Raw Shred Stream 解码器 + Decoded Shred Stream 转发器**,适配 [shredstream.com](https://www.shredstream.com) 的 UDP 推送。

接收 shredstream.com 推送的原始 shred(每个 UDP 数据报一个 shred,与 leader 广播字节完全一致),完成解析、Reed-Solomon 纠删恢复、Entry 流式反序列化,解码出交易后按官方 Decoded ShredStream 帧格式转发到你配置的 IP:端口 —— 官方各语言客户端(`decoded-shredstream-rust/js/go/python`)可直接消费。

```
                     ┌─────────────────────────────────────────────────┐
 UDP raw shreds      │  接收线程 (可绑核)                                │
 (shredstream.com ───►  解析 → slot 门控 → 每 slot 累积器               │
  分配的端口)         │   ├─ 流式 Entry 解码 (最低延迟路径)               │
                     │   ├─ Reed-Solomon 恢复丢失 data shred            │  有界队列
                     │   └─ 卡批 50ms 强制终结 + 抢救性解码               ├──────────►
                     └─────────────────────────────────────────────────┘            │
                                                                                      ▼
                                                                              转发线程
                                                                              UDP: 官方帧,每交易一报
                                                                              TCP: u32 长度前缀 + 帧,自动重连
```

## 构建

```bash
cargo build --release
```

- 仓库附带 `.cargo/config.toml`,在 crates.io 访问缓慢时自动走 rsproxy.cn 镜像;如需官方源删除该文件即可。
- 默认使用纯 Rust 的 Reed-Solomon 实现(无需 C 编译器)。生产环境(x86_64 Linux)可启用 SIMD 加速:
  ```bash
  cargo build --release --features simd    # 需要 gcc/clang
  ```

## 配置

默认读取当前目录 `config.toml`,也可 `shred-decode <path>` 指定:

```toml
[listener]
bind = "0.0.0.0"
port = 8001                  # shredstream.com dashboard 分配给你的 UDP 端口
recv_buffer_bytes = 67108864 # 64 MiB SO_RCVBUF
busy_poll_us = 200           # SO_BUSY_POLL(仅 Linux);删除或设 0 禁用
pool_buffers = 4096          # 2 KiB 接收缓冲池
max_slot_age = 3             # slot 状态保留窗口
# pin_cpu = 0                # 可选:接收线程绑核(仅 Linux)

[decoder]
enable_fec = true                # Reed-Solomon 恢复
max_fec_sets_per_slot = 32
stuck_batch_timeout_ms = 50      # 卡住的批超时强制终结
disable_salvage_delivery = false # true 则不投递损坏批的抢救交易(p99 更优)
queue_capacity = 8192            # 解码 → 转发 有界队列;满则丢新批并计数

[[outputs]]                      # 可配置多个输出
protocol = "udp"                 # udp | tcp
host = "127.0.0.1"
port = 9000

[stats]
interval_secs = 10
```

## 输出格式

### UDP(与官方 Decoded ShredStream UDP 推送一致)

一个数据报 = 一帧,小端:

| 偏移 | 长度 | 字段 | 值 |
|---|---|---|---|
| 0 | 2 | magic | `0x5AE7` |
| 2 | 1 | version | `2` |
| 3 | 1 | msg_type | `2`(transactions) |
| 4 | 1 | flags | `0` |
| 5 | 1 | frag_index | `0` |
| 6 | 1 | frag_count | `1` |
| 7 | 1 | padding | `0` |
| 8 | 8 | seq | 数据报序号(每个输出独立递增) |
| 16 | … | payload | slot(u64 LE)+ 原始 Solana wire 交易字节 |

交易字节即 `VersionedTransaction` 的 bincode/wyncode 兼容序列化,消费端可直接 `bincode::deserialize`(Rust)或 `VersionedTransaction.deserialize`(JS)。每帧 ≤ 1408 字节,交易永不分片。

### TCP

同一帧内容,前面加 `u32 LE` 长度(本项目的流式分帧约定,便于可靠消费)。断线自动重连(100ms→5s 退避),重连期间帧计数丢弃并在统计里体现。

## 运行与验证

```bash
# 1) 启动解码转发器
./target/release/shred-decode config.toml

# 2) 本地起一个接收端观察输出
./target/release/mock-sink 9000
# up=2s frames=32 (24/s) txs=32 slot=100002 seq=32 gaps=0 bad=0

# 3) 无真实订阅时,用合成 shred 打通链路
./target/release/examples/synth_shreds --udp 127.0.0.1:8001 --slots 3

# 4) 离线回放真实抓包(每行一个数据报的十六进制)
./target/release/synth_shreds --hex dump.txt --slots 3   # 生成样本
./target/release/replay dump.txt
# slot=100000 txs=8 first_sig=a0a0a0a0a0a0a0a0..
# summary: datagrams=18 batches=6 txs=48 bad_lines=0
```

抓取真实数据:在服务器上 `tcpdump -i any -w shreds.pcap udp port <你的端口>`,用 `tshark -T fields -e data.data` 导出 hex 后喂给 `replay` 即可离线验证解码正确性。

## 测试

```bash
cargo test
```

25 项测试覆盖:shred 解析偏移黄金测试、变体分类、wincode 流式 Entry 解码、**合成 shred 端到端解码**、**丢片 Reed-Solomon 恢复**、乱序到达、多批 slot、**卡批超时抢救**、缓冲池回收、官方帧编解码往返、**引擎→UDP 转发端到端**。

## 性能设计

- 单接收线程完成 解析→累积→解码 全流水线(与官方 SDK 同思路,免队列唤醒延迟);转发独立线程,经 crossbeam 有界队列衔接,反压时丢新批计数、绝不阻塞接收。
- 零拷贝解析:shred 解析只借用接收缓冲池中的字节;2048B 定长缓冲循环复用,稳态零分配。
- 流式 Entry 解码:shred 到达即推进反序列化,不等整批凑齐 —— 延迟最低路径;恢复批走一次性解码。
- `wincode`(bincode 兼容)反序列化 + `rustc_hash` + BTreeMap 有序累积;Reed-Solomon 矩阵按 (data, coding) 组合 LRU 缓存。
- slot 门控(slot < 2^40、前跳 ≤ 1M、已淘汰 slot 黑名单)与 `max_age=3` 滚动淘汰,尾部收割兜底。

Linux 生产环境建议:

```bash
# 提升 UDP 接收缓冲上限(否则 64MiB 请求会被内核钳制)
sysctl -w net.core.rmem_max=67108864
# 可选:接收线程绑核(config.toml 里 pin_cpu)
# busy_poll 已默认开启(200µs),需要 NIC 与内核支持
```

## 线格式参考(输入:原始 shred,小端)

```
[0x00..0x40)  前缀(重传/修复元数据,解码不需要)
[0x40]        variant 字节: 0xA5 数据legacy | 0x5A 纠删legacy
              0x60-0x6F/0x70-0x7F CodeMerkle(低4位=proof_size, 后者resigned)
              0x90-0x9F/0xB0-0xBF DataMerkle(同上)
[0x41] slot u64   [0x49] index u32   [0x4D] version u16   [0x4F] fec_set u32
数据 shred: [0x53] parent_offset u16  [0x55] flags(0b0100_0000=批完成, 0b1100_0000=slot末尾)
            [0x56] size u16           payload = raw[0x58..size]
纠删 shred: [0x53] num_data u16  [0x55] num_coding u16  [0x57] position u16
纠删分片长度 = 1228 - 89 - (proof_size*20 + 32 + resigned*64)
批(FEC set)payload = u64 LE entry 数 + 连续序列化的 Entry{num_hashes u64, hash 32B, Vec<VersionedTransaction>}
```

注意:与官方 SDK 一致,本项目**不校验 leader 签名**(流本身来自付费认证通道),仅做结构校验与 slot 合理性门控。

## 目录结构

```
src/variant.rs     变体分类          src/accumulator.rs  每 slot 累积/批状态机
src/parser.rs      shred 零拷贝解析   src/engine.rs       slot 门控与淘汰
src/fec.rs         Reed-Solomon 恢复  src/listener.rs     UDP 接收循环
src/decoder.rs     wincode 流式解码   src/forwarder.rs    UDP/TCP 转发
src/frame.rs       官方输出帧编解码   src/pool.rs         缓冲池
src/bin/mock_sink.rs  接收端测试工具  src/bin/replay.rs   离线回放
examples/synth_shreds.rs  合成 shred 生成器(压测/联调)
```
