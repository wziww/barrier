### 应用层之下

##### 测试环境

- driver: r8169 
- version: 2.3LK-NAPI

#### 驱动 pci 模块声明部分

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

```c
/* file kernel/drivers/net/ethernet/realtek/r8169.c
 * 接上文 probe 中 rtl_init_one 函数
 * 在该函数中需要关心的主要是一个 struct net_device *dev;    // net_device 结构，它抽象了对应的网络设备
 * dev.netdev_ops 里面放置的函数即为一个网络设备的操作方法集
 * dev->netdev_ops = &rtl_netdev_ops;
 */
static const struct net_device_ops rtl_netdev_ops = {
  .ndo_open		= rtl_open,      // 网络设备激活，状态变更为 up 时进行调用  ifconfig eth0 up
  .ndo_stop		= rtl8169_close, // 网络设备 down 的时候进行调用    			 ifconfig eth0 down
  .ndo_get_stats64	= rtl8169_get_stats64, // 系统通过该函数进行网卡驱动 stats 数据获取，否则通过 dev->stats 获取
  .ndo_start_xmit		= rtl8169_start_xmit,  
  /*  网络协议栈发送数据时，调用驱动中该函数向硬件发送数据
   *  ndo_start_xmit向硬件提交数据后立即返回
   *  硬件在完成发送后，以中断的方式通知OS\
   *  中断服务程序通过内核api——netif_wake_queue (ndev)通知协议栈可再度调用ndo_start_xmit
   */
  .ndo_tx_timeout		= rtl8169_tx_timeout,
  .ndo_validate_addr	= eth_validate_addr,
  .ndo_change_mtu		= rtl8169_change_mtu,
  .ndo_fix_features	= rtl8169_fix_features,
  .ndo_set_features	= rtl8169_set_features,
  .ndo_set_mac_address	= rtl_set_mac_address,
  .ndo_do_ioctl		= rtl8169_ioctl,
  .ndo_set_rx_mode	= rtl_set_rx_mode,
#ifdef CONFIG_NET_POLL_CONTROLLER
  .ndo_poll_controller	= rtl8169_netpoll,
#endif

};
```

