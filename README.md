# Pyverbs学习
learning https://github.com/linux-rdma/rdma-core/blob/master/Documentation/pyverbs.md  
device name ('mlx4_0')  

# Jupyter Notebook
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



