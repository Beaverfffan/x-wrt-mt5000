# 03 · x-wrt 适配清单

> 目标基线：x-wrt master（≥ 2026-09-16，含 YT9224 合入）+ kernel 6.18
> 上游来源：openwrt/openwrt PR #24237 头提交 `996b7d38`

## A. 设备支持（从 PR 移植，基本原样）

| # | 内容 | 来源 | x-wrt 落点 | 注意 |
|---|---|---|---|---|
| A1 | `mt7987a-gl-mt5000.dts` | PR | `target/linux/mediatek/dts/` | reset-gpio 已在最新修订改为 `pio 48`（GL 原厂固件实测定案）；gmac0=conduit(gdm/2500base-x)、gmac1=WAN(xgdm/internal phy) |
| A2 | filogic.mk 设备定义 | PR | `target/linux/mediatek/image/filogic.mk` | eMMC 设备，factory.bin + sysupgrade-tar |
| A3 | `02_network`：`ucidef_set_interfaces_lan_wan "lan1 lan2" eth1` | PR | filogic/base-files |  |
| A4 | `platform.sh` sysupgrade 名单 | PR | filogic/base-files |  |
| A5 | filogic `config-6.18` 增加 `CONFIG_NET_DSA_REALTEK*=` 三项 | PR | filogic/config-6.18 |  |
| A6 | generic `config-6.18` 增加 `# CONFIG_NET_DSA_REALTEK_RTL8366UB is not set` / `# CONFIG_NET_DSA_TAG_RTL8366UB_8021Q is not set` | PR 评论（dmtpascoal 定案） | target/linux/generic/config-6.18 | 否则其它 target syncconfig 遇到未回答的 Kconfig 提示直接挂（PR CI 全红的原因） |

## B. 驱动与 tagger（从 PR 移植，少量刷新）

| # | 内容 | 来源 | 说明 |
|---|---|---|---|
| B1 | `795-10-net-dsa-realtek-add-rtl8366ub.patch`（3617 行新 DSA 驱动） | PR | 直接进 `target/linux/generic/pending-6.18/`。驱动 `get_tag_protocol()` 返回 `RTL8366UB_8021Q`，注册 `dsa_tag_8021q_register(ds, ETH_P_8021Q)`，并关闭原生 RTL8_4 CPU tag；port isolation 使所有 LAN 流量经 CPU（LAN↔LAN 也能被 PPE 接管） |
| B2 | `795-11` 的 dsa.h/Kconfig/Makefile/`tag_rtl8366ub_8021q.c` 部分 | PR | 协议号 **32**（x-wrt 上空闲，YT922X_4B=35）；tagger 与 `tag_8021q.c` 的错误路径修复原样可用 |
| B3 | `795-11` 的 `mtk_eth_soc.c` 两个 hunk（`TX_DMA_SPTAG_V3`、`MTK_GDMA_SPECIAL_TAG` 仅限 `DSA_TAG_PROTO_MTK`） | PR | **需要移植到 x-wrt 的 mtk_eth_soc.c**：x-wrt 750/751 已改过该文件，patch 需 refresh 后核对上下文 |
| B4 | `795-12` dummy NAPI 初始化提前 | PR | 与 x-wrt 750/751 可能重复，先核对再决定取舍 |

## C. x-wrt 适配层（⚠️ 本研究核心，PR 的对应改动是死代码）

PR 795-11 对 `mtk_ppe_offload.c` 的改动（`mtk_flow_get_dsa_port()` 加 RTL8366UB_8021Q case、
`push_vid` 选择逻辑）**在 x-wrt 上不进入编译**（natflow 用 `mtk_ppe_offload1.o`），
必须按下表重放到 995 补丁：

| # | 位置（995 补丁内） | 改动 |
|---|---|---|
| C1 | `net/dsa/user.c` `dsa_flow_offload_check()` | 新增 `case DSA_TAG_PROTO_RTL8366UB_8021Q`：`dsa_port_is_vlan_filtering(dp)` → `-EOPNOTSUPP`；`dp->index > 4` → `-EOPNOTSUPP`；vid = 在 bridge 中 ? `dsa_tag_8021q_bridge_vid(dsa_port_bridge_num_get(dp))` : `dsa_tag_8021q_standalone_vid(dp)`；`path->dsa_port = ((vid & 0x3ff) << 5) \| dp->index` |
| C2 | `mtk_ppe_offload1.c` `mtk_offload_get_dsa_info()` | 新增 `case DSA_TAG_PROTO_RTL8366UB_8021Q`：解包与 MXL 相同（拒绝 bit15、port 限 0..4）；port=0 合法（lan1 就是 port 0） |
| C3 | `mtk_ppe_offload1.c` `mtk_offload_prepare_v4()` / `_v6()` 两处 | `set_vlan` 分支条件加入 `DSA_TAG_PROTO_RTL8366UB_8021Q` |

> C1 的 bridge-aware VID 选择照抄 PR 795-11 `mtk_flow_get_dsa_port()` 的语义，
> 但搬到 check 阶段（x-wrt 架构里 VID 必须随 dsa_port 打包穿越，PPE 侧拿不到 user 口 dp）。

补丁编号建议：mediatek/patches-6.18 已用到 995（natflow），新补丁取 **996+**；
注意与既有 `996-net-dsa-mt7530-an8855-reset.patch` 撞号——x-wrt 现网 996/997/998 已被占用，
从 **999** 开始（与 flint4 适配同一惯例），文件名如
`999-0001-net-mediatek-ppe1-offload-rtl8366ub-8021q.patch`。

## D. 构建与刷机

- 编译机：内网 Ubuntu `192.168.15.157`（用户 beaver，曾用于构建 PR 分支，路径 `~/openwrt-mt5000`）
- 流程参照 xwrt-flint4-adaptation：`tools/apply-to-xwrt.sh` 幂等装补丁 →
  `make -j$(nproc)` → 产物核对（`mtk_ppe_offload1.o` 体积/grep 两处站点/镜像 sha256）→ 刷机

## E. 风险与开放问题

| 风险 | 影响 | 缓解 |
|---|---|---|
| PR 未合并（blocked），驱动大（3617 行），维护者倾向扩展 upstream realtek 驱动 | 上游基础可能变动 | 锁定 `996b7d38` 快照；关注 dmtpascoal/gl-mt5000-openwrt 替代驱动 |
| x-wrt MXL case 只打包 standalone VID，桥接场景行为未验证 | bpi r4 pro 参照机型上需先实测 MXL 路径 | 首次真机验证含"br-lan 桥接 + iperf3"用例；我们的 case 从第一天 bridge-aware |
| x-wrt yt922x PPE 自评 "Hardware remains untested" | PPE1 的 YT922X_4B 分支未经过硬件 | 不影响 RTL8366UB 路径，但说明该代码新鲜度高，需完整回归 |
| MT7987 + natflow PPE 真机验证情况未知 | SoC 新 | 首刷先做基础连通 + hwnat=0 对照，再开 hwnat=1 |
| natflow 升级改动 `*1.c` 基线 | 补丁可能打不上 | 打不上即报错（不静默），按 docs/02 重放 |
| 795-12 与 x-wrt 750/751 重复/冲突 | 编译失败 | 移植 B3/B4 时统一 refresh 后核对 |

## F. 里程碑

1. **M1 设备点亮**：A 组移植完成，x-wrt 固件启动，lan1/lan2/eth1 连通（hwnat=0）
2. **M2 PPE 适配**：B/C 组完成，编译产物核对通过
3. **M3 真机验证**：docs/04 三证据全部通过，LAN↔LAN 与 WAN↔LAN 均硬件转发
4. **M4 回馈**：整理补丁集，视情况向 x-wrt（ptpt52）提 PR、向 PR #24237 反馈 x-wrt 适配经验
