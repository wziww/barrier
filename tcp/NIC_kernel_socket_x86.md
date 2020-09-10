### 应用层之下

##### 测试环境

- driver: r8169 
- version: 2.3LK-NAPI

```c
/* file: kerner/drivers/net/ethernet/eraltek/r8169.c
 */
static struct pci_driver rtl8169_pci_driver = {
  .name		= MODULENAME,				 // 驱动名 可通过 ethtool -i xxx 来确认
  .id_table	= rtl8169_pci_tbl, // 驱动 id 表，会根据驱动内的这部分信息对应到相关设备生产厂家，设备型号之类的信息去校验，当匹配上了内核会认为这个驱动服务能为此设备进行服务
  .probe		= rtl_init_one,    // 驱动探测函数（初始化）
  .remove		= rtl_remove_one,  // 从内核中移除设备时触发，如果不是热拔插设备，这边的函数为 NULL
  .shutdown	= rtl_shutdown,    // 设备关闭时触发
  .driver.pm	= RTL8169_PM_OPS,
};
/* 这边涉及一个名词 PCI
 * PCI是一个总线标准，PCI总线上的设备就是PCI设备，这些设备有很多类型，当然也包括网卡设备
 * 在平时查看系统中断统计的时候，也经常能看见 PCI 相关的统计项
 * cat /proc/interrupts （例）
 * 26:    1861609          4   PCI-MSI-edge      enp3s0
 */
module_pci_driver(rtl8169_pci_driver);
```

> ethtool : query or control network driver and hardware settings
>
> 网络驱动及相关硬件查看及设置工具


```shell
➜  ~ ethtool -i enp3s0
driver: r8169
version: 2.3LK-NAPI
firmware-version: rtl8168g-2_0.0.1 02/06/13
expansion-rom-version:
bus-info: 0000:03:00.0
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: yes
supports-priv-flags: no
```

