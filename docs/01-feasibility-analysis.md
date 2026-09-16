# 01 · 可行性分析：MT5000 的 RTL8366UB 能否用 YT9224 / MxL862 同款方式上 PPE 硬件加速

> 分析日期：2026-09-16
> 依据：openwrt/openwrt PR #24237 头提交 `996b7d38`、x-wrt master `8373cff1`、
> Linux 6.18 上游源码、Beaverfffan/xwrt-flint4-adaptation（docs 分支）

## 1. 硬件与拓扑

GL-MT5000（Brume 3）：

- SoC：MediaTek MT7987A（四核 A53 @ 2.0GHz），x-wrt 已有 SoC 级支持
  （patches-6.18 的 360/361/740/741/750/751/752/821/831/844 覆盖 pinctrl/clk/PCS/eth/2.5G PHY/PWM/LVTS/cpufreq）
- LAN：2× 2.5G，挂在 **RTL8366UB** 交换机上，DSA CPU 口（port 17）经 2500base-x fixed-link 接 gmac0
- WAN：1× 2.5G，SoC 内置 PHY（phy15）接 gmac1
- DTS 关键片段（PR 快照见 `reference/pr-mt7987a-gl-mt5000.dts`）：
  `gmac0` 为 switch conduit（`mac-type = "gdm"`，2500base-x fixed-link），
  `gmac1` 为 WAN（`mac-type = "xgdm"`，internal phy）。

RTL8366UB 属于 Realtek 新一代（rtl8372 同代），与 upstream `rtl8366rb` 不是同一代芯片。

## 2. 问题本质：为什么原生 tag 上不了 PPE

GL.iNet 在 795-11 补丁的 commit message 里写得很清楚：

> Its native RTL8_4 tag hides the network-layer EtherType from the MediaTek PPE,
> and the PPE cannot insert that proprietary 8-byte tag. Consequently LAN-to-WAN
> packets cannot be parsed and WAN-to-LAN flow entries cannot target a DSA user port.

即：MTK PPE 硬件转发引擎只认识**标准 802.1Q 形状的 4 字节头**（vlan_layer=1 的
表项在 SA 后插入 `[etype][vlan1]` 共 4 字节）。任何私有 tag：

- 入方向：tag 把网络层 EtherType 藏掉 → PPE 解析不了流（LAN→WAN 方向无法绑定）
- 出方向：PPE 无法替硬件插私有 tag → 出口到交换机用户口的流无法指定目的端口

YT9224（原生 8 字节 tag）和 RTL8366UB（原生 8 字节 RTL8_4 tag）是同一个病。
flint4 的老方案是"让 PPE 出口发 YT9224 认识的 4 字节私有 tag"（802.1Q 形状 + 自定义 ctrl）；
GL.iNet 在 PR 里选择了更上游化的路：**直接给交换机用标准 802.1Q 管理 tag**。

## 3. 三个候选方案对比

### 方案 A：RTL8_4 原生 tag + PPE 出口重放（类比 flint4 老方案）

让 PPE 出口插入 RTL8_4 能解码的 tag。不可行：RTL8_4 是 **8 字节** tag，
PPE 一层 VLAN 层只有 4 字节，硬件插不出来。❌

### 方案 B：私有 4 字节 tag（flint4 的 yt922x_4b 方式）

给 RTL8366UB 配一个 4 字节私有 CPU tag（类似 yt922x_4b 的 802.1Q 形状 + 自定义 ctrl），
PPE 出口用 `mtk_foe_entry_set_vlan(ctrl)` 下发。技术上可行（flint4 已验证同一机制），
但要自定义 Realtek 的 tag 解码格式、写新 tagger、并维护私有编码。社区接受度低。⚠️

### 方案 C：标准 802.1Q tagger（tag_8021q 框架）—— ✅ 推荐

给 RTL8366UB 用内核现成的 `dsa_tag_8021q_register()` 框架（Vladimir Oltean 的通用
"管理 VLAN"方案），DSA 管理流量走标准 802.1Q tag，PPE 双向天然认识：

- **入方向**：lan 口进来的帧带标准 802.1Q tag → PPE 解析无遮挡
- **出方向**：PPE 表项用 `mtk_foe_entry_set_vlan(vid)` 把管理 VID 写进 VLAN TCI →
  交换机按 VLAN 表转发到目的端口

这正是 **bpi r4 pro 的 MxL862（`MXL862_8021Q`）** 用的机制，也是 GL.iNet PR 24237
已经实现并自测的方案（795-10 驱动注册 `dsa_tag_8021q_register(ds, ETH_P_8021Q)`，
795-11 提供 `RTL8366UB_8021Q` tagger + 上游 `mtk_ppe_offload.c` 支持）。
三个独立实现（上游 6.19+ 的 mxl862、bpi r4 pro 适配、GL.iNet MT5000 PR）收敛到同一条路，
说明这是该问题的"正解"。

## 4. x-wrt 的转发模型：为什么"短 tag"就够了

x-wrt 与官方 OpenWrt 的行为差异是理解整个方案的钥匙：

**x-wrt 故意屏蔽交换机自身的 bridge/offload，LAN 侧全部流量上 CPU 走 natflow。**
具体三段式：

1. **入方向**：交换机不做转发决策，只按管理 tag 把帧送上 CPU（port isolation 保证用户口只通 CPU 口）
2. **转发决策**：CPU 上的软件 bridge + natflow 决定每条流的出口
3. **出方向**：PPE 绑定流后，只需替硬件插入一个 **4 字节短 tag**（形态类似 MTK SDK 的
   special tag）——交换机按短 tag 里的端口/VLAN 信息把帧送到目的口，
   即 **Port → PPE → Port 硬转**

这个模型下对交换机 tag 的要求只有两条：**① PPE 硬件发得出（4 字节、802.1Q 形状）
② 交换机解得出**。

- `MXL862_8021Q`（bpi r4 pro）：管理 VID 编码在标准 802.1Q TCI 里 ✅
- `YT922X_4B`（flint4/GL-BE14000）：4 字节 802.1Q 形状 + 端口 ctrl ✅
- `RTL8366UB_8021Q`（MT5000，本方案）：tag_8021q 框架，同款短 tag ✅

**bpi r4 pro 是 MTK 官方推荐开发板**，其 mxl862 + 8021Q tagger 就是这套
"外部交换机 + PPE 加速"模型的 MTK 体系内参照实现——对齐它，等于对齐 MTK SDK 的思路，
这也是该方案社区/上游接受度最高的原因。

同时注意：因为 LAN↔LAN 也全过 CPU，**MT5000 的 LAN-LAN 吞吐同样依赖 PPE 生效**
（交换机内直转被 isolation 屏蔽）。验证时 LAN-LAN 必须和三证据一起看，
参照 flint4 实测：修复前 hwnat=1 时 0.09~0.15 Gbit/s（近断流），修复后 2.32~2.34 Gbit/s、
CPU 从 ~33% 降到 ~5%。

## 5. x-wrt PPE1 现状（关键）

x-wrt 用自己的 natflow 硬件加速栈，替换了 MediaTek 驱动的 PPE 实现：

- `mtk_ppe.o` / `mtk_ppe_offload.o` → **`mtk_ppe1.o` / `mtk_ppe_offload1.o`**
  （995-0001-hwnat-add-natflow-flow-offload-support.patch，2841 行，自含全部 PPE 源码）
- 凡是 `+++ b/drivers/net/ethernet/mediatek/mtk_ppe_offload.c` 的补丁在 x-wrt 上都是**死代码**
  （flint4 适配踩过的坑，见 xwrt-flint4-adaptation `docs/01`）

**好消息**：x-wrt 的 PPE1 今天就有完整的 802.1Q 出口 tag 支持：

| 组件 | 位置（995 补丁内） | 对 802.1Q 的支持 |
|---|---|---|
| `dsa_flow_offload_check()` | `net/dsa/user.c` 钩子（ndo_flow_offload_check） | `DSA_TAG_PROTO_MXL862_8021Q` case：把管理 VID 打包进 `path->dsa_port = ((vid & 0x3ff) << 5) \| index` |
| `mtk_offload_get_dsa_info()` | `mtk_ppe_offload1.c` | 按 conduit 实际 tag proto 解包：MXL → `*vid = 0xc00 \| ((port >> 5) & 0x3ff)`，port &= 0x1f |
| 出口 tag 写入 | `mtk_offload_prepare_v4/v6()` | `dsa_proto == MXL862_8021Q \|\| YT922X_4B` → `mtk_foe_entry_set_vlan(vid)`；否则 `set_dsa`（MTK 私有 tag） |

2026-09-16 x-wrt 刚合入 YT9224 支持（commit `6148c498` 驱动 + `5def9c44` PPE1 yt922x_4b case），
并**刻意把 DSA 协议号 32 留给 target-local tagger**（YT921X=33、YT922X=34、YT922X_4B=35，32 空缺）——
而 GL.iNet PR 的 `RTL8366UB_8021Q` 恰好用的就是 **32**。在 x-wrt 上零冲突。

VID 编码（`net/dsa/tag_8021q.c`）：RSV=0xC00（bits10-11 恒为 11），
standalone vid = `0xC00 | switch_id<<6 | port`，bridge vid = `0xC00 | VBID(1..7)`，
低 10 位足够编码 → 复用 MXL 的 `(vid & 0x3ff) << 5 | port` 打包方案没有位数问题。

## 6. 结论

**可以用同样的方式，且工作量小于 flint4 适配：**

1. 不需要自定义私有 4 字节 tag（方案 B 放弃）——用 tag_8021q 框架，与 bpi r4 pro / 上游收敛一致；
2. x-wrt PPE1 已有 MXL862_8021Q 同构路径，扩展点明确、改动小（两个 case + 一个分支条件）；
3. 协议号 32 在 x-wrt 已被刻意保留；
4. RTL8366UB 驱动本身做了 port isolation（user 口只通 CPU 口），与 x-wrt
   "全部流量走 natflow" 的模型天然吻合 → **LAN↔LAN 也能被 PPE 接管**
   （Port→PPE→Port 硬转，正是 x-wrt 关掉交换机 offload 想要的效果）；
5. 必须绕开的坑：PR 的 `mtk_ppe_offload.c` 改动是死代码，要按 flint4 的方法论
   重放到 995 补丁的 PPE1 + user.c 里，并以 `mtk_ppe_offload1.o` 产物和三证据验证为准。

**待确认项**（见 docs/03 风险节）：
- x-wrt MXL case 只打包 standalone vid，而 lan1/lan2 在 br-lan 里时 tag_8021q 会
  删除 standalone vid（bridge_join 时）→ 我们的实现必须像 PR 795-11 一样
  **bridge-aware** 地选 vid；
- x-wrt 的 yt922x PPE 支持自评 "Hardware remains untested"，mxl862 路径的真机状态
  需在 bpi r4 pro 上先行核对（作为机制参照）；
- MT7987 + natflow PPE 在 x-wrt 上的真机验证情况未知，需首刷确认。
