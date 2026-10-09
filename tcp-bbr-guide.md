# Linux TCP BBR：检查、启用、验证与回滚

[返回首页](README.md) · [性能诊断](network-performance.md)

> 本文为智连 Secure Access 品牌技术资料，不构成加速效果承诺。示例面向拥有管理权限的 Linux 主机；不适用于直接修改 Windows、macOS 或用户态 QUIC 的拥塞控制。

## 1. 先确认是否值得改

BBR 根据带宽和传播时延等观测建立发送模型。它是一种 TCP 拥塞控制选择，不会提高物理带宽，也不能修复 DNS、证书、错误路由或服务端限速。不同 BBR 实现、内核和队列环境可能表现不同；不能凭算法名称宣称固定提升比例。

设置主要影响该主机发出的 TCP 数据。服务器向客户端下载时，服务器是主要数据发送端；只改接收端并不会让远端发送端改用 BBR。代理终止 TCP 后，连接分段管理。单纯转发数据包的路由器也不会接管端点 TCP 的拥塞控制。

发行版可能将模块编入内核、作为模块提供或完全不提供。`bbr` 这个名字不足以识别具体 BBR 版本，必须结合内核来源和补丁记录。不建议为了本文替换生产内核或运行来历不明的一键脚本。

## 2. 只读检查与保存原值

```bash
uname -r
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control
sysctl net.core.default_qdisc
tc qdisc show
```

保存输出，并记下当前拥塞算法与实际网卡队列。容器通常共享宿主机内核，是否可修改由网络命名空间、权限和宿主策略决定；容器里失败不应通过盲目增加特权来解决。

以下变量仅保存在当前 shell；关闭终端前，把实际值另行保存在变更记录中：

```bash
previous_cc=$(sysctl -n net.ipv4.tcp_congestion_control)
previous_qdisc=$(sysctl -n net.core.default_qdisc)
printf 'previous_cc=%s\nprevious_qdisc=%s\n' "$previous_cc" "$previous_qdisc"
```

如果可用算法没有 `bbr`，且本机发行版确实提供模块，可执行：

```bash
sudo modprobe tcp_bbr
sysctl net.ipv4.tcp_available_congestion_control
```

若提示模块不存在，停止并查阅该发行版内核文档。`lsmod` 中看不到模块并不证明没有 BBR：它也可能直接编入内核。

## 3. 先做临时变更

确认可用算法包含 `bbr` 后：

```bash
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
sysctl net.ipv4.tcp_congestion_control
```

这个默认值用于新建连接；应用也可能自行设置套接字算法，已有连接不会因此统一切换。应重新建立测试连接，在持续传输时运行 `ss -tin`，定位正确的源/目的地址，观察算法名称。只看 sysctl 输出还不能证明业务连接已经使用 BBR。[Linux 内核 IP sysctl 文档](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)

`fq` 与发送节奏控制常一起用于 BBR 部署，但不是每台机器都应覆盖为相同队列。不同内核对 pacing 的实现存在差异。上游快速指南中的经典配置包含：

```bash
sudo sysctl -w net.core.default_qdisc=fq
```

这修改默认队列选择，**不代表现有网卡的活动队列立即变成 fq**。使用 `tc qdisc show` 核查。多队列网卡可能显示 `mq` 根队列和不同叶子队列，虚拟设备也可能为 `noqueue`。不要为追求输出一致而直接覆盖已有整形或调度策略。[BBR 上游快速指南](https://github.com/google/bbr/blob/master/Documentation/bbr-quick-start.md)

本指南不自动替换活动队列。若需要变更活动队列，应先保存完整 qdisc/class/filter 配置，理解父子关系，并安排恢复窗口。

## 4. 用业务连接做 A/B 对照

在同一端点、同一对象、相近时段，交替比较原算法和 BBR；每次切换后新建连接。固定并发和地址族，记录吞吐、负载时延、重传、失败样本、CPU 与内存。单次测速或只有多连接下载结果不足以判断。

同时观察交互业务是否受影响。如果吞吐没有稳定改善、延迟上升或出现兼容问题，应恢复原值。远端服务限速、应用读写不足或出口拥塞可能是主因；不要继续叠加未经验证的缓冲区参数。

## 5. 验证后再持久化

只有临时测试符合预期后，才创建专用配置文件。先检查 `/etc/sysctl.conf`、`/etc/sysctl.d/` 和发行版其他 sysctl 配置目录是否已有同名参数，避免多处定义互相覆盖。

以下是建议文件 `/etc/sysctl.d/90-zhilian-bbr.conf` 的内容，手动确认文件不存在或备份后再写入：

```ini
# 经本机测试后保留；记录变更日期和原算法
net.ipv4.tcp_congestion_control = bbr
# 仅在本机队列策略已评估后另行启用：
# net.core.default_qdisc = fq
```

可单独加载该文件，避免无意加载其他未审查配置：

```bash
sudo sysctl -p /etc/sysctl.d/90-zhilian-bbr.conf
```

如果需要启动时加载模块，按发行版模块加载机制配置；不要假定所有系统都会自动完成。维护窗口重启后再次检查默认算法、实际连接和队列，确认启动配置最终生效。

## 6. 回滚

在保存原变量的同一 shell 中：

```bash
sudo sysctl -w net.ipv4.tcp_congestion_control="$previous_cc"
sudo sysctl -w net.core.default_qdisc="$previous_qdisc"
```

如果终端已关闭，使用变更记录里的真实原值，不要猜测原算法必定是 `cubic`。删除本次专用文件中的新增参数，或恢复备份；只操作本次创建的配置，不删除其他服务的文件。若新增了模块自动加载配置，也一并撤销。

回滚默认队列值不会恢复被手工改过的活动队列；若另行执行过 `tc` 变更，还需根据保存的队列配置恢复。最后新建业务连接复测，确认问题消失。通常不需要强行卸载正在使用的模块。

## 7. 验证边界

本文命令依据上游资料整理，未在你的服务器执行，也未产生任何智连线路性能数据。实际部署前请核对发行版支持、维护窗口与恢复入口。
