# tcpfit

按每台机器实测推导的 TCP 调优工具. 不套用固定参数, 实测 BDP 与限速器拐点.

本脚本由 [kylin010](https://github.com/Kylin010) 编写；本 fork 增加了供应链、SSH 与隐私加固.

## 安装

```bash
git clone https://github.com/uiuiqkjn/tcpfit.git
cd tcpfit
sha256sum -c SHA256SUMS
sudo ./install.sh
```

安装器只使用已经下载并校验的本地文件，不支持 `curl | bash`，也不会以 root 身份二次下载代码。
完成后运行 `sudo tcpfit`，选 1 可进入全自动流程.

## 三种用法

| 用法 | 命令 |
|---|---|
| 安装 | `sha256sum -c SHA256SUMS && sudo ./install.sh` |
| 装好后 | `sudo tcpfit` |
| 子命令 | `sudo tcpfit tune --role proxy --bw 500` |

## 菜单

```
   1. 一键调优   Auto-tune (recommended)  ~10 min
   2. 基础调优   Base tuning only          ~1 min
   3. 拐点测试   Policer sweep             ~8 min
   4. 加 swap    Add swap (low-memory box)
   ────────────────────────────────────────────
   5. 查看状态   Status
   6. 端口验证   Verify port capability    ~1 min
   7. 回滚改动   Rollback all changes
   8. 检查更新   Check for updates
   9. 调优存档   Tuning archives
   u. 卸载 tcpfit
```

脚本不会自动更新或替换自身. 菜单 8 或 `tcpfit update` 只检查版本并显示 release 地址；
升级时重新下载完整 release、校验 `SHA256SUMS`、审阅变更后运行本地安装器.

## 隐私与联网

运行计数默认关闭。只有显式设置 `TCPFIT_TELEMETRY=1`，或创建
`/var/lib/tcpfit/telemetry-enabled` 后，菜单启动才会请求
`https://tcpfit.spacevps.cc/ping?v=<版本>`。应用层只发送版本号，但服务端仍能看到来源 IP、
时间及 HTTP/TLS 元数据。删除标记文件并取消环境变量即可再次关闭。

测速会连接用户选择的 iperf3 对端；选择公共节点时，相应服务商会看到来源 IP 和测试流量。

一键调优只问三个问题: 带宽、测速对端、机器用途. 确认之后跑到底不再打断.

带宽那一问支持四种输入:

| 输入 | 行为 |
|---|---|
| 数字 | 按该带宽推导缓冲区, 然后实测拐点 |
| 回车 | 现场实测带宽, 然后实测拐点 |
| `m` | 直接填限速值, 跳过拐点扫描 |
| `0` | 不做整形 |

## 子命令

```bash
tcpfit detect                                     # 机器画像
tcpfit probe    --peer <近处iperf3服务器>          # 探测可用带宽
tcpfit tune     --role proxy --bw 500             # 基础调优
tcpfit tune     --role proxy --bw 500 --save 换机房前   # 调优并给存档命名
tcpfit sweep    --peer <近处iperf3服务器> --nominal 500
tcpfit shape    --rate 510                        # 应用整形
tcpfit shape    --off                             # 移除整形, 保留基础调优
tcpfit harden   --swap 2G                         # 加 swap
tcpfit verify   --peer <近处iperf3服务器>          # 测速验证
tcpfit status                                     # 当前配置
tcpfit rollback                                   # 回滚全部改动
tcpfit update                                     # 检查更新
tcpfit archive list                               # 列出存档
tcpfit archive save "晚间配置"                     # 保存当前状态
tcpfit archive restore 0010                       # 恢复第 10 份存档
tcpfit archive rename 0010 "备用配置"              # 存档改名
tcpfit archive show   0010                        # 查看某份存档的内容
tcpfit archive delete 0010                        # 删除指定存档（别名 rm）
tcpfit uninstall --keep-archives                  # 卸载，保留存档和快照
```

## PPPoE / 拨号线路

家宽 PPPoE、以及任何默认路由长这样的机器：

```
default dev ppp0 scope link          # 点对点, 没有 via
```

0.5.8 起支持. 这类机器每次重拨都是新接口, qdisc 和路由窗口会跟着消失,
所以 tcpfit 会往 `/etc/ppp/ip-up.d/50-tcpfit` 放一个钩子, 让整形和 initcwnd
在拨通后自动回来. 钩子只对调优时那块网卡生效, 不会碰机器上别的 ppp 链路.

没有 `/etc/ppp/ip-up.d` 的机器不受影响, 不会被创建任何东西.

## 多机（未上线）

多机编排还没在真实环境验证过, 暂时不建议使用. 下面的用法仅供参考.

编排器现在强制验证 SSH 主机密钥。第一次连接前，请通过服务商控制台等可信渠道核对主机
指纹，并将它加入 `~/.ssh/known_hosts`；不能通过验证时会直接停止，不会静默接受新密钥。
优先使用 SSH 密钥。兼容的密码字段通过匿名管道交给 `sshpass`，不会出现在命令行参数中。


```bash
cp inventory/servers.example.yml inventory/servers.yml
chmod 600 inventory/servers.yml
vi inventory/servers.yml

python3 orchestrator/fleet.py detect
python3 orchestrator/fleet.py tune
python3 orchestrator/fleet.py sweep
python3 orchestrator/fleet.py shape --auto
python3 orchestrator/fleet.py verify
```

选项: `--only 机器名` `--tag 标签` `-j 并发数` `--dry-run`.
临时执行任意命令: `fleet.py run -- uptime`.

## 它改了什么

| 类别 | 参数 |
|---|---|
| 拥塞控制 | `tcp_congestion_control=bbr` + `default_qdisc=fq` |
| 缓冲区 | `tcp_rmem` / `tcp_wmem` / `rmem_max` / `wmem_max` / `tcp_mem` |
| 窗口 | `tcp_window_scaling` / `tcp_moderate_rcvbuf` / `tcp_adv_win_scale` |
| 队列 | `netdev_max_backlog` / `netdev_budget` / `somaxconn` 等 |
| 连接 | `tcp_tw_reuse` / `tcp_fin_timeout` / `ip_local_port_range` 等 |
| 起步 | `tcp_slow_start_after_idle=0` / `initcwnd 32` |
| 出向整形 | HTB 全局上限 + fq 叶子 pacing |

共 32 个 sysctl 参数. 缓冲区和整形值按每台机器实测推导, 不是固定值.

## 拐点扫描怎么工作

先不限速跑一次, 看有没有东西在打你:

| 结果 | 动作 |
|---|---|
| 丢包低 | 没有限速器, 不整形 |
| 丢包高 | 有限速器, 从实测吞吐往上扫找拐点 |
| 吞吐 > 2500 Mbit | 超出扫描上限, 不扫（可用 `--cap` 调整） |

拐点在"不限速吞吐"的**上面** —— 打穿限速器会让吞吐掉下来, 所以往上找.

## 回滚

```bash
tcpfit rollback                # 按快照逐项写回, 不是恢复默认
tcpfit rollback --purge-swap   # 同时删掉 harden 建的 /swapfile
tcpfit shape --off    # 只去掉整形
```

首次改动前自动存快照到 `/var/lib/tcpfit/pre-tune.snapshot`, 记录全部 32 项参数的原始值.

0.5.7 起，原始快照同时保留为 `0000 出厂状态`，不可改名或单独删除。
`archive restore 0000` 与 `rollback` 使用同一回滚流程；“出厂状态”指首次调优前的快照。
普通存档位于 `/var/lib/tcpfit/archives/`。基础调优后自动保存，一键调优则在最终整形、验证完成后保存。
序号可以输入 `10` 或 `0010`；名字含空格时请加引号。

恢复普通存档会同步 sysctl 启动配置和整形服务。有路由窗口设置时沿用 networkd-dispatcher hook；
缺少该目录会提示路由只能即时恢复并返回失败。恢复失败可能已经应用部分设置，请按提示检查后重试。

`tcpfit uninstall` 默认删除存档；需要保留则加 `--keep-archives`。
若回滚失败，卸载会停止并保留存档。卸载不删除 swap、iperf3 或 ping。

swap 默认不动 —— 删掉正在用的 swap 可能让机器立刻 OOM, 要一并撤销得显式加 `--purge-swap`.

改动只落在这些文件, 不碰 `/etc/sysctl.conf`:

```
/etc/sysctl.d/99-tcpfit.conf
/etc/systemd/system/tcpfit-qdisc.service
/usr/local/sbin/tcpfit-qdisc.sh
/etc/networkd-dispatcher/routable.d/50-tcpfit-initcwnd
/etc/modules-load.d/tcpfit-bbr.conf
/var/lib/tcpfit/
```

用了 `harden --swap` 还会创建 `/swapfile` 并往 `/etc/fstab` 加一行 —— 这两个 `rollback` 默认不动,
要一并撤销加 `--purge-swap`. 缺 iperf3 时经你确认后会用包管理器安装它.

## 已知限制

- 瓶颈在国际链路而非端口时, 整形不会带来提升, 但输出看起来一切正常
- 扫满区间没找到拐点时会把区间上界当成拐点, 这种情况用 `m` 手动指定
- 需要 Linux + systemd + iproute2. OpenVZ/LXC 上 `tc` 和 `initcwnd` 可能受限
- `sweep` 需要一台近处的 iperf3 对端

## 从 nettune 升级

老机器上的产物文件名还是 `nettune-*`, 新版本会自动检测并搬迁, 快照和 rollback 都保留. 直接跑新版即可.

## 许可证

[MIT](LICENSE)
