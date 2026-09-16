# 02 · x-wrt natflow PPE1 的 DSA 卸载链路解剖

> 依据：x-wrt master `8373cff1` 的
> `target/linux/mediatek/patches-6.18/995-0001-hwnat-add-natflow-flow-offload-support.patch`
> （2841 行，自含 `mtk_ppe1.c` / `mtk_ppe_offload1.c` / user.c 钩子）
> 快照：`reference/xwrt-995-natflow.patch`

## 1. 总览：一个流表项从 netfilter 到 PPE 的路

```
nftables flowtable (natflow 接管)
  └─ flow_offload_hw_path_t src / dest        ← natflow 自己的 offload API（flow_offload_t）
       └─ dest 经过 DSA user 口时：
            net/dsa/user.c  dsa_flow_offload_check()   [ndo_flow_offload_check 钩子]
              ├─ 校验 tag proto，决定能否卸载（default: -EOPNOTSUPP）
              ├─ 把管理 VID 打包进 path->dsa_port
              └─ path->dev 从 user 口换成 conduit（eth0）
       └─ mtk_ppe_offload1.c  mtk_flow_offload_add()
            ├─ mtk_offload_prepare_v4() / _v6()
            │    ├─ mtk_offload_get_dsa_info()  ← 按 conduit 实际 tag proto 解包 dsa_port
            │    ├─ mtk_foe_entry_set_queue()   ← DSA 队列映射
            │    └─ 出口 tag：set_vlan(vid) 或 set_dsa(port)
            └─ 下发 FOE 表项 → PPE 硬件接管
```

**关键架构点**：跨过 `dsa_flow_offload_check()` 之后，`dest->dev` 已经变成 conduit，
用户口身份只能经由 `dsa_port` 的位打包传下去。这就是为什么 x-wrt 把 VID 塞进 `dsa_port`
而不是像上游 `mtk_ppe_offload.c` 那样在 PPE 侧现算（上游在 `mtk_flow_get_dsa_port()`
里能拿到 `dsa_user_to_port(dev)`，因为上游的 check 路径不替换 dev，见
`reference/kernel-6.18-mtk_ppe_offload.c`）。

## 2. `dsa_flow_offload_check()`（995 补丁内, net/dsa/user.c）

```c
static int dsa_flow_offload_check(flow_offload_hw_path_t *path)
{
	struct net_device *dev = path->dev;
	struct dsa_port *dp;

	if (!(path->flags & FLOW_OFFLOAD_PATH_ETHERNET))
		return -EINVAL;

	dp = dsa_user_to_port(dev);
	/* Validate before changing dev: some callers only inspect the path. */
	switch (dp->cpu_dp->tag_ops->proto) {
	case DSA_TAG_PROTO_MXL862_8021Q:
		if (!dp->index || dp->index > 16)
			return -EOPNOTSUPP;
		/* The outer VID's top two bits are always 11. */
		path->dsa_port = ((dsa_tag_8021q_standalone_vid(dp) & 0x3ff) << 5) |
				 dp->index;
		break;
	case DSA_TAG_PROTO_YT922X_4B:
		/* The software tagger supplies access-port PVIDs. */
		if (dsa_port_is_vlan_filtering(dp) || dp->index > 8)
			return -EOPNOTSUPP;
		path->dsa_port = dp->index;
		break;
	case DSA_TAG_PROTO_MTK:
		if (dp->index >= 32)
			return -EOPNOTSUPP;
		path->dsa_port = dp->index;
		break;
	default:
		return -EOPNOTSUPP;
	}
	path->dev = dsa_user_to_conduit(dev);
	path->flags |= FLOW_OFFLOAD_PATH_DSA;

	if (path->dev->netdev_ops->ndo_flow_offload_check)
		return path->dev->netdev_ops->ndo_flow_offload_check(path);

	return 0;
}
```

- 这是**协议白名单**：不在名单内的 tag proto 一律拒绝卸载（流量回落软件快转，不会出错包）。
- MXL 打包格式：`dsa_port[14:5] = vid[9:0]`，`dsa_port[4:0] = port`，bit15 恒 0。

## 3. `mtk_offload_get_dsa_info()`（mtk_ppe_offload1.c）

```c
/* Decode against the conduit's actual tag protocol, not packed VID bits. */
static int
mtk_offload_get_dsa_info(flow_offload_hw_path_t *dest, u16 *port, u16 *vid,
                         enum dsa_tag_protocol *proto)
{
	*port = dest->dsa_port;
	*vid = 0;
	*proto = DSA_TAG_PROTO_NONE;
	if (*port == 0xffff)
		return 0;

	struct dsa_port *cpu_dp = dest->dev->dsa_ptr;
	const struct dsa_device_ops *tag_ops;

	if (!cpu_dp) return -EOPNOTSUPP;
	tag_ops = READ_ONCE(cpu_dp->tag_ops);
	if (!tag_ops) return -EOPNOTSUPP;

	*proto = tag_ops->proto;
	switch (*proto) {
	case DSA_TAG_PROTO_MXL862_8021Q:
		/* Reject the old encoding and out-of-range physical ports. */
		if ((*port & BIT(15)) || !(*port & 0x1f) ||
		    (*port & 0x1f) > 16)
			return -EINVAL;
		*vid = 0xc00 | ((*port >> 5) & 0x3ff);
		*port &= 0x1f;
		return 0;
	case DSA_TAG_PROTO_YT922X_4B:
		if (*port > 8)
			return -EINVAL;
		*vid = dsa_tag_yt922x_4b_port(*port);
		return 0;
	case DSA_TAG_PROTO_MTK:
		return *port < MTK_DSA_USER_PORT_MAX ? 0 : -EINVAL;
	default:
		return -EOPNOTSUPP;
	}
}
```

## 4. 出口 tag 写入（`mtk_offload_prepare_v4/v6()`，两处对称）

```c
	if (dsa_port != 0xffff) {
		if (dsa_port < MTK_DSA_USER_PORT_MAX) {
			...
			mtk_foe_entry_set_queue(eth, entry, mac->dsa_queue_base + mac->dsa_port_rank[dsa_port]);
		}
		if (dsa_proto == DSA_TAG_PROTO_MXL862_8021Q ||
		    dsa_proto == DSA_TAG_PROTO_YT922X_4B)
			mtk_foe_entry_set_vlan(eth, entry, dsa_vid);
		else
			mtk_foe_entry_set_dsa(eth, entry, dsa_port);
	}
```

- `mtk_foe_entry_set_vlan(vid)`：置 vlan_layer=1 + vlan_tag，把 16 位 TCI **原样**写进
  `l2->vlan1` → 硬件发出的 4 字节头 = `81 00 <TCI>`，交换机按 VLAN 表转发。
- `mtk_foe_entry_set_dsa(port)`：`l2->etype = BIT(port)`（MediaTek 私有 CPU tag 格式），
  只有 MT7530/MT7531 类交换机认识。

## 5. RTL8366UB_8021Q 需要插入的位置

完全对称的两处扩展（详见 docs/03）：

1. `dsa_flow_offload_check()` 加 `case DSA_TAG_PROTO_RTL8366UB_8021Q`，
   **bridge-aware** 地选 VID（在 bridge 里用 `dsa_tag_8021q_bridge_vid(bridge_num)`，
   standalone 用 `dsa_tag_8021q_standalone_vid(dp)`；`dsa_port_is_vlan_filtering(dp)` 拒绝卸载），
   按 MXL 同款格式打包；端口校验 `dp->index <= 4`（RTL8366UB 用户口 0..4，MT5000 用 0/1）。
2. `mtk_offload_get_dsa_info()` 加 `case DSA_TAG_PROTO_RTL8366UB_8021Q` 解包
   （位运算与 MXL 完全相同，仅端口上限放宽到允许 port 0），
   并把该 proto 加入第 4 节的 `set_vlan` 分支条件。

## 6. 为什么 vid 必须 bridge-aware（bridge_join 删 standalone vid）

`net/dsa/tag_8021q.c`：

```c
dsa_tag_8021q_bridge_join():
	dsa_port_tag_8021q_vlan_add(dp, bridge_vid, true);
	dsa_port_tag_8021q_vlan_del(dp, standalone_vid, false);   /* ← standalone VID 被删 */
```

MT5000 的 lan1/lan2 默认在 br-lan 里。如果 PPE 出口仍写 standalone VID，
交换机 VLAN 表里该端口已不属于这个 VID → 帧被丢弃（就是 flint4 修复前的
"绑定正常、 silently 断流"故障形态）。x-wrt 现有 MXL case 只打包 standalone VID，
**桥接场景下需要实测确认**；我们的新 case 从第一天就按 PR 795-11 的 bridge-aware
逻辑实现，规避此风险。

## 7. VID 位域（tag_8021q.c，打包方案的位数依据）

```
bit 11-10  RSV = 0b11          → 0xC00
bit  8-6   switch_id (0..7)
bit  5-4,9 VBID (bridge 1..7)
bit  3-0   port (0..15)
```

所有管理 VID 的低 10 位已含全部可变信息 → `(vid & 0x3ff) << 5 | port` 打包无损。
