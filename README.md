# x-wrt-mt5000

把 **GL.iNet GL-MT5000 (Brume 3)** 适配到 [x-wrt](https://github.com/x-wrt/x-wrt)，并让它的
**Realtek RTL8366UB 交换机在 MTK PPE（x-wrt natflow hwnat）下跑满硬件加速**。

> 硬件：MT7987A SoC（4×A53 2.0GHz）+ 2×2.5G LAN（RTL8366UB 交换机）+ 1×2.5G WAN（SoC 内置 PHY）

## 一句话结论（可行性分析）

**可以，而且比 YT9224 的适配更顺。** GL.iNet 在上游 PR
[openwrt/openwrt#24237](https://github.com/openwrt/openwrt/pull/24237) 里已经放弃了
RTL8366UB 原生 8 字节 RTL8_4 tag（"PPE 无法插入私有 8 字节 tag"），改为写一个基于内核
`tag_8021q` 框架的 `RTL8366UB_8021Q` tagger——这正是 **bpi r4 pro 的 MxL862 所用
`MXL862_8021Q` 的同款机制**。而 x-wrt 的 natflow PPE1（`mtk_ppe_offload1.c`）
**今天就有 MXL862_8021Q 的现成出口 VLAN tag 路径**（2026-09-16 又刚合入 YT922X_4B），
为 RTL8366UB_8021Q 增加 case 是完全同构的扩展。

旧的 flint4 适配（纯私有 4 字节 tag + PPE 出口 tag 重放）在本案**不需要**：
802.1Q tagger 方案与上游、bpi r4 pro、x-wrt 三边的收敛方向一致，接受度最高。

⚠️ 唯一的坑和 flint4 相同：PR 的 PPE 补丁改的是上游 `mtk_ppe_offload.c`，
**在 x-wrt 上是死代码**（natflow 编译的是 `mtk_ppe_offload1.o`），必须重放到
x-wrt 的 995 natflow 补丁里。详见 [docs/01](docs/01-feasibility-analysis.md)。

## 文档

| 文件 | 内容 |
|---|---|
| [docs/01-feasibility-analysis.md](docs/01-feasibility-analysis.md) | **可行性分析**：三个方案对比、x-wrt 机制解剖、结论 |
| [docs/02-xwrt-ppe1-mechanism.md](docs/02-xwrt-ppe1-mechanism.md) | x-wrt natflow PPE1 的 DSA 卸载链路逐层说明（dsa_port 打包/解包） |
| [docs/03-adaptation-plan.md](docs/03-adaptation-plan.md) | x-wrt 适配清单：每处改动、补丁落点、风险 |
| [docs/04-verification-plan.md](docs/04-verification-plan.md) | 真机验证方案（吞吐 + CPU + PPE 表项三证据）与判定阈值 |

## 参考快照（reference/）

分析所依据的源码快照（GPL-2.0）：

- `pr-795-10-rtl8366ub-driver.patch` / `pr-795-11-rtl8366ub-8021q-ppe.patch` /
  `pr-mt7987a-gl-mt5000.dts` —— GL.iNet PR #24237 头提交 `996b7d38`
- `xwrt-995-natflow.patch` / `xwrt-799-40-yt922x-4b.patch` / `xwrt-YT9224.md` —— x-wrt master（含 2026-09-16 的 YT9224 合入）
- `kernel-6.18-*.c` / `8021q.h` / `tag_8021q.c` —— Linux 6.18 对应上游文件

## 相关仓库

- 旧方案（本案不采用，但验证方法论沿用）：[Beaverfffan/xwrt-flint4-adaptation](https://github.com/Beaverfffan/xwrt-flint4-adaptation)
- 上游 PR：[openwrt/openwrt#24237](https://github.com/openwrt/openwrt/pull/24237)
- x-wrt：[x-wrt/x-wrt](https://github.com/x-wrt/x-wrt)
- 社区替代 DSA 驱动：[dmtpascoal/gl-mt5000-openwrt](https://github.com/dmtpascoal/gl-mt5000-openwrt)、[gl-inet/gl-rtl8366ub](https://github.com/gl-inet/gl-rtl8366ub)

## License

GPL-2.0-only（补丁作用于 Linux 内核源码）。
