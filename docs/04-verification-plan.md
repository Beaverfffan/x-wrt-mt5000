# 04 · 真机验证方案

> 方法论沿用 xwrt-flint4-adaptation（`docs/07-verification.md`）：
> **吞吐 + CPU + PPE 表项三证据**，缺一不可。单看 iperf3 数字无法区分
> "软件快转"与"硬件卸载"。

## 1. 前置检查

```sh
# natflow 必须是作者出厂默认（不要套用任何「绕过」脚本）
uci show natflow | grep -E 'ifname_group|hwnat'
#   期望：ifname_group_type='0'、无 ifname_group 列表、hwnat='1'

# 确认编译进去的是 PPE1（不是上游 mtk_ppe_offload.o）
ls -la /sys/kernel/debug/ppe1/        # ppe1 存在即 natflow 栈
```

## 2. 编译产物核对（刷机前）

```sh
# 1. 对象文件重建且包含新逻辑（grep 源码树不算数，要看 build_dir 落盘文件）
grep -c DSA_TAG_PROTO_RTL8366UB_8021Q \
  build_dir/target-*/linux-*/linux-*/drivers/net/ethernet/mediatek/mtk_ppe_offload1.c
#   期望 ≥3（get_dsa_info case + v4/v6 两处分支条件）
grep -c DSA_TAG_PROTO_RTL8366UB_8021Q \
  build_dir/target-*/linux-*/linux-*/net/dsa/user.c
#   期望 ≥1

# 2. 镜像 sha256 与基线对比必须变化
```

## 3. 三证据验证（刷机后）

### 证据一：吞吐

```sh
# LAN 客户端(2.5G) --lan1--> MT5000 --lan2--> LAN 服务器(2.5G)，正向+反向各 3 次
iperf3 -c <server> -t 30 -P 4
```

判定：修复后 LAN↔LAN ≥ 2.2 Gbit/s（2.5G 口线速量级）。
若 ~0.1 Gbit/s 且重传爆炸 → 出口 tag 错误形态（flint4 故障的复现特征）。

### 证据二：CPU

```sh
top / htop 记录 iperf3 期间 CPU  busy
```

判定：同吞吐下 hwnat=1 的 CPU 应显著低于 hwnat=0（flint4 参照：33% → 5%）。
CPU 不降 = 软件快转，不算成功。

### 证据三：PPE 表项（最硬的证据）

```sh
cat /sys/kernel/debug/ppe1/entries | grep BND
```

期望形态（出口为 RTL8366UB 口时）：

```
BND ... etype=0008  vlan=<管理VID>      ← 正确：etype=载荷ethertype(IPv4 0x0800 字节序翻转打印为 0008)，
                                            vlan 低 10 位 + 0xC00 = standalone 或 bridge 管理 VID
BND ... etype=2000  vlan=0,0            ← 故障：etype=BIT(port) 是 MTK 私有 tag 格式
```

VID 对照（MT5000：switch_id=0，lan1=port0，lan2=port1）：

| 场景 | VID | 说明 |
|---|---|---|
| standalone lan1 | 0xC00+0 = 3072 | `vlan=3072` |
| standalone lan2 | 0xC00+1 = 3073 | |
| br-lan 内 lan1/lan2 | 0xC00+VBID(1) = 3088 | VBID=1<<4 |

### WAN↔LAN 方向

外网下载打流，同时抓 PPE 表项与 CPU；回流方向（WAN→LAN，出口 lan 口）的
出口 tag 必须是上述 802.1Q 形态。

## 4. 回归矩阵

| 用例 | 期望 |
|---|---|
| hwnat=0 LAN↔LAN | 线速（软件快转基线） |
| hwnat=1 LAN↔LAN | 线速 + CPU 大幅下降 + 表项 vlan=管理VID |
| hwnat=1 WAN→LAN 下载 | 线速 + 同上 |
| hwnat=1 LAN→WAN 上传 | 线速 + 同上 |
| br-lan 桥接场景（默认配置） | 同上线速（bridge VID 正确下发） |
| VLAN-filtering 子接口 | 该流拒绝卸载（-EOPNOTSUPP 回落软件），不丢包 |
| dmesg | 无 natflow/DSA/PPE 报错，无 -EOPNOTSUPP 刷屏 |

## 5. 失败定位速查

| 症状 | 大概率原因 |
|---|---|
| PPE 表项为空 | dsa_flow_offload_check 拒绝（proto 不在白名单）→ 查 user.c case 是否生效 |
| 表项 etype=BIT(port)、vlan=0 | set_vlan 分支条件漏加 RTL8366UB_8021Q |
| 表项对但断流/近断流 | VID 错（standalone vs bridge）→ 交换机 VLAN 表无此端口；对照第 3 节 VID 表 |
| 吞吐对但 CPU 不降 | 软件快转，表项其实没绑定成功 |
