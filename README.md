# Pyverbs学习
learning https://github.com/linux-rdma/rdma-core/blob/master/Documentation/pyverbs.md  
device name ('mlx4_0')，是ConnectX‑3 (MLX4)，链路层Ethernet，跑ROCEv2。  
现状限制：  
1. 标准 IB Verbs API 全部可用：Context / PD / MR / QP / CQ / SRQ 都正常
2. **mlx4dv 私有扩展彻底不可用**：`Mlx4DVContextAttr` / Mlx4Context 这类 DV 专属接口不存在，rdma‑core 上游已经删除 mlx4dv 用户态库，没有 python binding。
3. MLX4 硬件本身 DV 能力就残缺：不支持很多现代 ROCEv2 调优、硬件 offload，没有 packet pacing、MPRQ、timestamp 等。
4. （new）**MLX4 (CX‑3) pyverbs AH 不支持全局 GRH AH 对象**。
导致后果：  
1. **Jupyter 里绝对不要用 `with` 管理 RDMA 资源**，全部手动创建、手动按顺序释放
2. **释放顺序严格从子到父**：AH → MR → QP → CQ → PD → Context，顺序反了必崩

# md代码实操
## 打开并查询IB设备
```python
import pyverbs.device as d
ctx = d.Context(name='mlx4_0')

attr = ctx.query_device()
print(attr)
```
```bash
FW version            : 2.42.5000           
Node guid             : e41d:2d03:0018:9f40 
Sys image GUID        : e41d:2d03:0018:9f43 
Max MR size           : 0xffffffffffffffff  
Page size cap         : 0xfffffe00          
Vendor ID             : 0x2c9               
Vendor part ID        : 4103                
HW version            : 0                   
Max QP                : 425912              
Max QP WR             : 16351               
Device cap flags      : 
  IBV_DEVICE_BAD_PKEY_CNTR
  IBV_DEVICE_BAD_QKEY_CNTR
  IBV_DEVICE_CHANGE_PHY_PORT
  IBV_DEVICE_UD_AV_PORT_ENFORCE
  IBV_DEVICE_PORT_ACTIVE_EVENT
  IBV_DEVICE_SYS_IMAGE_GUID
  IBV_DEVICE_RC_RNR_NAK_GEN
  IBV_DEVICE_MEM_WINDOW
  IBV_DEVICE_UD_IP_CSUM
  IBV_DEVICE_XRC
  IBV_DEVICE_MEM_MGT_EXTENSIONS
  IBV_DEVICE_MEM_WINDOW_TYPE_2B
  IBV_DEVICE_RAW_IP_CSUM
  IBV_DEVICE_MANAGED_FLOW_STEERING
  
Max SGE               : 32                  
Max SGE RD            : 30                  
MAX CQ                : 65408               
Max CQE               : 4194303             
Max MR                : 524032              
Max PD                : 32764               
Max QP RD atom        : 16                  
Max EE RD atom        : 0                   
Max res RD atom       : 6814592             
Max QP init RD atom   : 128                 
Max EE init RD atom   : 0                   
Atomic caps           : 1                   
Max EE                : 0                   
Max RDD               : 0                   
Max MW                : 0                   
Max raw IPv6 QPs      : 0                   
Max raw ethy QP       : 0                   
Max mcast group       : 131072              
Max mcast QP attach   : 244                 
Max AH                : 2147483647          
Max FMR               : 0                   
Max map per FMR       : 8191                
Max SRQ               : 65472               
Max SRQ WR            : 16383               
Max SRQ SGE           : 31                  
Max PKeys             : 128                 
local CA ack delay    : 15                  
Phys port count       : 1                   
```
## 查询GID
```python
import pyverbs.device as d
ctx = d.Context(name='mlx4_0')
gid = ctx.query_gid(port_num=1, index=3)
print(gid)
```
```bash
0000:0000:0000:0000:0000:0000:0000:0000
```

## 查询端口
```python
import pyverbs.device as d
ctx=d.Context(name='mlx4_0')
port_attr = ctx.query_port(1)
print(port_attr)
```
```bash
Port state              : Active (4)          
Max MTU                 : 4096 (5)            
Active MTU              : 4096 (5)            
SM lid                  : 0                   
Port lid                : 0                   
lmc                     : 0x0                 
Link layer              : Ethernet            
Max message size        : 0x40000000          
Port cap flags          : 
  IBV_PORT_CM_SUP
  IBV_PORT_IP_BASED_GIDS
  
Port cap flags 2        : 
                   
max VL num              : 2                   
Bad Pkey counter        : 0                   
Qkey violations counter : 0                   
GID table len           : 128                 
Pkey table len          : 1                   
SM sl                   : 0                   
Subnet timeout          : 0                   
Init type reply         : 0                   
Active width            : 4X (2)              
Active speed            : 5.0 Gbps (4)        
Phys state              : Link up (5)         
Flags                   : 1                  
```
## 扩展查询设备
```python
import pyverbs.device as d
ctx = d.Context(name='mlx4_0')
attr = ctx.query_device_ex()
attr.max_dm_size
```
```bash
0
```
```python
attr.rss_caps.max_rwq_indirection_table_size
```
```bash
256
```
## 创建PD
```python
import pyverbs.device as d
from pyverbs.pd import PD

with d.Context(name='mlx4_0') as ctx:
    pd = PD(ctx)
```

## 创建MR
```python
import pyverbs.device as d
from pyverbs.pd import PD
from pyverbs.mr import MR
from pyverbs.libibverbs_enums import ibv_access_flags

with d.Context(name='mlx4_0') as ctx:
    with PD(ctx) as pd:
        mr_len = 1000
        flags = ibv_access_flags.IBV_ACCESS_LOCAL_WRITE
        mr = MR(pd, mr_len, flags)
```
## 创建MW（不支持）
## 创建DM（不支持）
## 创建DMMR（不支持）
## 创建CQ
```python
import random

from pyverbs.cq import CompChannel, CQ
import pyverbs.device as d

with d.Context(name='mlx4_0') as ctx:
    num_cqes = random.randint(0, 200)
    comp_vector = 0
    if random.choice([True, False]):
        with CompChannel(ctx) as cc:
            cq = CQ(ctx, num_cqes, None, cc, comp_vector)
    else:
        cq = CQ(ctx, num_cqes, None, None, comp_vector)
    print(cq)
```
```bash
CQ
Handle                : 0                   
CQEs                  : 127     
```
## 创建extended CQ（不支持）
## 创建AH
创建GlobalRoute, AHAttr, AH，在RoCE环境运行
```python
from pyverbs.addr import GlobalRoute, AHAttr, AH
import pyverbs.device as d
from pyverbs.pd import PD

with d.Context(name='mlx4_0') as ctx:
    port_number = 1
    gid_index = 0
    gid = ctx.query_gid(port_number, gid_index)
    gr = GlobalRoute(dgid=gid, sgid_index=gid_index)
    ah_attr = AHAttr(gr=gr, is_global=1, port_num=port_number)
    print(ah_attr)
    with PD(ctx) as pd:
        ah = AH(pd, attr=ah_attr)
```
```bash
  port num              : 1                   
  sl                    : 0                   
  source path bits      : 0                   
  dlid                  : 0                   
  static rate           : 0                   
  is global             : 1                   
  前面是多出来的
  dgid                  : fe80:0000:0000:0000:e61d:2dff:fe18:9f40
  flow label            : 0                   
  sgid index            : 0                   
  hop limit             : 1                   
  traffic_class         : 0
```
## 创建QP（进度目前在这里）

### 附录（RoCEv1）

### 代码
```python
from pyverbs.qp import QPCap, QPInitAttr, QPAttr, QP
from pyverbs.addr import GlobalRoute
from pyverbs.addr import AH, AHAttr
import pyverbs.device as d
from pyverbs.libibverbs_enums import ibv_qp_type
from pyverbs.pd import PD
from pyverbs.cq import CQ
import pyverbs.wr as pwr

ctx = d.Context(name='mlx4_0')
pd = PD(ctx)
cq = CQ(ctx, 100, None, None, 0)
cap = QPCap(100, 10, 1, 1, 0)
qia = QPInitAttr(cap=cap, qp_type = ibv_qp_type.IBV_QPT_UD, scq=cq, rcq=cq)

udqp = QP(pd, qia, QPAttr())
port_num = 1
gid_index = 3
gid = ctx.query_gid(port_num, gid_index)
gr = GlobalRoute(dgid=gid, sgid_index=gid_index)
ah_attr = AHAttr(gr=gr, is_global=1, port_num=port_num)
ah=AH(pd, ah_attr)
wr = pwr.SendWR()
wr.set_wr_ud(ah, 0x1101, 0)
udqp.post_send(wr)
```
```bash
传参错误
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
/tmp/ipykernel_1341720/1019242947.py in <module>
     20 gr = GlobalRoute(dgid=gid, sgid_index=gid_index)
     21 ah_attr = AHAttr(gr=gr, is_global=1, port_num=port_num)
---> 22 ah=AH(pd, ah_attr)
     23 wr = pwr.SendWR()
     24 wr.set_wr_ud(ah, 0x1101, 0)

addr.pyx in pyverbs.addr.AH.__init__()

TypeError: __init__() takes exactly 1 positional argument (2 given)
```
## 创建extended QP（确定不支持）

## 创建XRCD（仅限原生IB）

## 创建XRC SRQ（不仅限原生IB）

## 后面mlx5的provider操作（mlx4dv私有扩展彻底不可用）
上游 rdma‑core 已经废弃 mlx4dv，pyverbs 新版本已经删掉 mlx4 provider  
mlx5 provider的pyverbs.providers.mlx5.mlx5dv没有pyverbs.providers.mlx4.mlx4dv  
## 打开
## 查询
## 创建QP
## 创建CQ

## 创建CMID（这个不应该失败。支持 RoCEv1、RoCEv2；但是不支持 iWARP。需要进一步尝试）
失败
## ParentDomain（MemAlloc内存对齐）
## MLX5 VAR
## 数据包调速PP
## 用户访问区域UAR
## 导入 设备 PD MR
## 流量导向Flow Steering
## 以太网参数规格项Ethernet spec
## MLX5 DevX对象
控制pd？


# 附录
## 关键信息
1. HCA：MLX4103 ConnectX-3 Pro，RoCEv1，ens1，IP `192.168.2.24`
2. GID0：`fe80::e61d:2dff:fe18:9f40`（link-local，RoCEv1）
GID1：`::ffff:192.168.2.24`（IPv4 mapped GID，RoCEv1，**开发优先用这个 GID**）
3. 网卡二层 MTU 9000；RDMA HCA MTU 固定 4096，UD WR payload 最大要扣掉 RoCE 头。
4. **无 ODP，buffer 必须 mlock**
5. 内核模块 ok；但你手动编译 rdma-core 的 lib 路径污染 LD_LIBRARY_PATH，导致 ibstat 挂掉。ibv_devinfo 不受影响。
6. RoCEv1 限制：**只能同二层广播域，不能跨网段**，对端机器必须在 192.168.2.0/24 同网段。
## 网络扫描结果
全网段
```
192.168.2.1（网关，不用管）
192.168.2.24（本机）
192.168.2.27
192.168.2.28
192.168.2.30
192.168.2.31
192.168.2.35
192.168.2.115
192.168.2.125
192.168.2.126
192.168.2.129
192.168.2.150
192.168.2.202
192.168.2.203
192.168.2.220
```
带网卡
```
192.168.2.28
192.168.2.30
192.168.2.31
192.168.2.35
192.168.2.115
192.168.2.126
```
## GID
```bash
show_gids
```
```bash
DEV     PORT    INDEX   GID                                     IPv4            VER     DEV
---     ----    -----   ---                                     ------------    ---     ---
mlx4_0  1       0       fe80:0000:0000:0000:e61d:2dff:fe18:9f40                 v1      ens1
mlx4_0  1       1       0000:0000:0000:0000:0000:ffff:c0a8:0218 192.168.2.24    v1      ens1
n_gids_found=2
```
## 查询设备信息
```bash
ibv_devinfo -v
```
```bash
hca_id: mlx4_0
        transport:                      InfiniBand (0)
        fw_ver:                         2.42.5000
        node_guid:                      e41d:2d03:0018:9f40
        sys_image_guid:                 e41d:2d03:0018:9f43
        vendor_id:                      0x02c9
        vendor_part_id:                 4103
        hw_ver:                         0x0
        board_id:                       MT_1100111019
        phys_port_cnt:                  1
        max_mr_size:                    0xffffffffffffffff
        page_size_cap:                  0xfffffe00
        max_qp:                         425912
        max_qp_wr:                      16351
        device_cap_flags:               0x2d7e9c66
                                        BAD_PKEY_CNTR
                                        BAD_QKEY_CNTR
                                        CHANGE_PHY_PORT
                                        UD_AV_PORT_ENFORCE
                                        PORT_ACTIVE_EVENT
                                        SYS_IMAGE_GUID
                                        RC_RNR_NAK_GEN
                                        MEM_WINDOW
                                        UD_IP_CSUM
                                        XRC
                                        MEM_MGT_EXTENSIONS
                                        MEM_WINDOW_TYPE_2B
                                        RAW_IP_CSUM
                                        MANAGED_FLOW_STEERING
                                        Unknown flags: 0x8488000
        max_sge:                        32
        max_sge_rd:                     30
        max_cq:                         65408
        max_cqe:                        4194303
        max_mr:                         524032
        max_pd:                         32764
        max_qp_rd_atom:                 16
        max_ee_rd_atom:                 0
        max_res_rd_atom:                6814592
        max_qp_init_rd_atom:            128
        max_ee_init_rd_atom:            0
        atomic_cap:                     ATOMIC_HCA (1)
        max_ee:                         0
        max_rdd:                        0
        max_mw:                         0
        max_raw_ipv6_qp:                0
        max_raw_ethy_qp:                0
        max_mcast_grp:                  131072
        max_mcast_qp_attach:            244
        max_total_mcast_qp_attach:      31981568
        max_ah:                         2147483647
        max_fmr:                        0
        max_srq:                        65472
        max_srq_wr:                     16383
        max_srq_sge:                    31
        max_pkeys:                      128
        local_ca_ack_delay:             15
        general_odp_caps:
        rc_odp_caps:
                                        NO SUPPORT
        uc_odp_caps:
                                        NO SUPPORT
        ud_odp_caps:
                                        NO SUPPORT
        xrc_odp_caps:
                                        NO SUPPORT
        completion timestamp_mask:                      0x0000ffffffffffff
        hca_core_clock:                 427000kHZ
        raw packet caps:
                                        Scatter FCS offload
        device_cap_flags_ex:            0x2D7E9C66
        tso_caps:
                max_tso:                        131072
                supported_qp:
                                        SUPPORT_RAW_PACKET
        rss_caps:
                max_rwq_indirection_tables:                     425912
                max_rwq_indirection_table_size:                 256
                rx_hash_function:                               0x1
                rx_hash_fields_mask:                            0x800000FF
                supported_qp:
                                        SUPPORT_RAW_PACKET
        max_wq_type_rq:                 425912
        packet_pacing_caps:
                qp_rate_limit_min:      0kbps
                qp_rate_limit_max:      0kbps
        tag matching not supported

        cq moderation caps:
                max_cq_count:   65535
                max_cq_period:  65535 us

        num_comp_vectors:               96
                port:   1
                        state:                  PORT_ACTIVE (4)
                        max_mtu:                4096 (5)
                        active_mtu:             4096 (5)
                        sm_lid:                 0
                        port_lid:               0
                        port_lmc:               0x00
                        link_layer:             Ethernet
                        max_msg_sz:             0x40000000
                        port_cap_flags:         0x04010000
                        port_cap_flags2:        0x0000
                        max_vl_num:             2 (2)
                        bad_pkey_cntr:          0x0
                        qkey_viol_cntr:         0x0
                        sm_sl:                  0
                        pkey_tbl_len:           1
                        gid_tbl_len:            128
                        subnet_timeout:         0
                        init_type_reply:        0
                        active_width:           4X (2)
                        active_speed:           10.0 Gbps (4)
                        phys_state:             LINK_UP (5)
                        GID[  0]:               fe80:0000:0000:0000:e61d:2dff:fe18:9f40, RoCE v1
                        GID[  1]:               0000:0000:0000:0000:0000:ffff:c0a8:0218, RoCE v1

```
## 查询设备属性
```python
import pyverbs.device as d

ctx = d.Context(name='mlx4_0')
dev_attr = ctx.query_device()

# 打印完整属性表（和官方文档格式一致）
print("=== 完整设备属性 ===")
print(dev_attr)

# 打印所有可用属性名，确认准确字段
print("\n=== 所有属性名列表 ===")
print([attr for attr in dir(dev_attr) if not attr.startswith('_')])
```
```bash
=== 完整设备属性 ===
FW version            : 2.42.5000           
Node guid             : e41d:2d03:0018:9f40 
Sys image GUID        : e41d:2d03:0018:9f43 
Max MR size           : 0xffffffffffffffff  
Page size cap         : 0xfffffe00          
Vendor ID             : 0x2c9               
Vendor part ID        : 4103                
HW version            : 0                   
Max QP                : 425912              
Max QP WR             : 16351               
Device cap flags      : 
  IBV_DEVICE_BAD_PKEY_CNTR
  IBV_DEVICE_BAD_QKEY_CNTR
  IBV_DEVICE_CHANGE_PHY_PORT
  IBV_DEVICE_UD_AV_PORT_ENFORCE
  IBV_DEVICE_PORT_ACTIVE_EVENT
  IBV_DEVICE_SYS_IMAGE_GUID
  IBV_DEVICE_RC_RNR_NAK_GEN
  IBV_DEVICE_MEM_WINDOW
  IBV_DEVICE_UD_IP_CSUM
  IBV_DEVICE_XRC
  IBV_DEVICE_MEM_MGT_EXTENSIONS
  IBV_DEVICE_MEM_WINDOW_TYPE_2B
  IBV_DEVICE_RAW_IP_CSUM
  IBV_DEVICE_MANAGED_FLOW_STEERING
  
Max SGE               : 32                  
Max SGE RD            : 30                  
MAX CQ                : 65408               
Max CQE               : 4194303             
Max MR                : 524032              
Max PD                : 32764               
Max QP RD atom        : 16                  
Max EE RD atom        : 0                   
Max res RD atom       : 6814592             
Max QP init RD atom   : 128                 
Max EE init RD atom   : 0                   
Atomic caps           : 1                   
Max EE                : 0                   
Max RDD               : 0                   
Max MW                : 0                   
Max raw IPv6 QPs      : 0                   
Max raw ethy QP       : 0                   
Max mcast group       : 131072              
Max mcast QP attach   : 244                 
Max AH                : 2147483647          
Max FMR               : 0                   
Max map per FMR       : 8191                
Max SRQ               : 65472               
Max SRQ WR            : 16383               
Max SRQ SGE           : 31                  
Max PKeys             : 128                 
local CA ack delay    : 15                  
Phys port count       : 1                   

=== 所有属性名列表 ===
['atomic_caps', 'device_cap_flags', 'fw_version', 'hw_ver', 'local_ca_ack_delay', 'max_ah', 'max_cq', 'max_cqe', 'max_ee', 'max_ee_init_rd_atom', 'max_ee_rd_atom', 'max_fmr', 'max_map_per_fmr', 'max_mcast_grp', 'max_mcast_qp_attach', 'max_mr', 'max_mr_size', 'max_mw', 'max_pd', 'max_pkeys', 'max_qp', 'max_qp_init_rd_atom', 'max_qp_rd_atom', 'max_qp_wr', 'max_raw_ethy_qp', 'max_raw_ipv6_qps', 'max_rdd', 'max_res_rd_atom', 'max_sge', 'max_sge_rd', 'max_srq', 'max_srq_sge', 'max_srq_wr', 'node_guid', 'page_size_cap', 'phys_port_cnt', 'set_log_level', 'sys_image_guid', 'vendor_id', 'vendor_part_id']
```
## 查询CQ属性
```python
import pyverbs.device as d
from pyverbs.cq import CQ

ctx = d.Context(name='mlx4_0')
cq = CQ(ctx, 1024)
print("CQ 对象所有公开属性：")
print([attr for attr in dir(cq) if not attr.startswith('_')])
```
```bash
CQ 对象所有公开属性：
['ack_events', 'close', 'comp_channel', 'cq', 'cqe', 'poll', 'req_notify', 'resize', 'set_log_level']
```


## （格式相关）
## 创建
```python

```
```bash

```