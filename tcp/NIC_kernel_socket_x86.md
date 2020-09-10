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
  /*  网络协议栈发送数据时，调用驱动中该函数向硬件发送数据
   *  ndo_start_xmit向硬件提交数据后立即返回
   *  硬件在完成发送后，以中断的方式通知OS
   *  中断服务程序通过内核api——netif_wake_queue (ndev)通知协议栈可再度调用ndo_start_xmit
   */
  .ndo_start_xmit		= rtl8169_start_xmit,  
  .ndo_tx_timeout		= rtl8169_tx_timeout,
  /*
   * 1.watchdog_timeo
   * 用于实现传出超时的时间设定。
   * 2/ndo_tx_timeout
   * 在发送队列停止(netif_queue_stopped(dev)返回1)，且watchdog_timeo到期的时候，内核网络子系统会调用ndo_tx_timeout来进行处理
   */
  .ndo_validate_addr	= eth_validate_addr, // 测试多媒体访问地址是否有效
  .ndo_change_mtu		= rtl8169_change_mtu,  // 使用这个函数该表最大传输单元，如果没有定义，那么任何请求改变MTU的请求都会报错
  .ndo_fix_features	= rtl8169_fix_features,
  .ndo_set_features	= rtl8169_set_features,
  .ndo_set_mac_address	= rtl_set_mac_address, // 调用设置 MAC 地址
  .ndo_do_ioctl		= rtl8169_ioctl,             // 当用户请求一个 ioctl,如果 ioctl 没有相对应的接口函数，则返回not supported error code 
  .ndo_set_rx_mode	= rtl_set_rx_mode,				 // 通过该函数设置了单播过滤，没有实现这个函数的设备，由系统决定是否开启混杂模式，并且将值保存在net_device->uc_promisc中
#ifdef CONFIG_NET_POLL_CONTROLLER
  .ndo_poll_controller	= rtl8169_netpoll,
#endif

};
```

#### rtl_open

```c
static int rtl_open(struct net_device *dev)
{
  struct rtl8169_private *tp = netdev_priv(dev);
  void __iomem *ioaddr = tp->mmio_addr;
  struct pci_dev *pdev = tp->pci_dev;
  int retval = -ENOMEM;

  pm_runtime_get_sync(&pdev->dev);

  /*
   * Rx and Tx descriptors needs 256 bytes alignment.
   * dma_alloc_coherent provides more.
   */
  // tx ring buffer
  tp->TxDescArray = dma_alloc_coherent(&pdev->dev, R8169_TX_RING_BYTES,
                                       &tp->TxPhyAddr, GFP_KERNEL);
  if (!tp->TxDescArray)
    goto err_pm_runtime_put;
	// rx ring buffer
  tp->RxDescArray = dma_alloc_coherent(&pdev->dev, R8169_RX_RING_BYTES,
                                       &tp->RxPhyAddr, GFP_KERNEL);
  if (!tp->RxDescArray)
    goto err_free_tx_0;
  
  retval = rtl8169_init_ring(dev);
  if (retval < 0)
    goto err_free_rx_1;

  INIT_WORK(&tp->wk.work, rtl_task);

  smp_mb();

  rtl_request_firmware(tp);

  retval = request_irq(pdev->irq, rtl8169_interrupt,
                       (tp->features & RTL_FEATURE_MSI) ? 0 : IRQF_SHARED,
                       dev->name, dev);
  if (retval < 0)
    goto err_release_fw_2;

  rtl_lock_work(tp);

  set_bit(RTL_FLAG_TASK_ENABLED, tp->wk.flags);

  napi_enable(&tp->napi);

  rtl8169_init_phy(dev, tp);

  __rtl8169_set_features(dev, dev->features);

  rtl_pll_power_up(tp);

  rtl_hw_start(dev);

  if (!rtl8169_init_counter_offsets(dev))
    netif_warn(tp, hw, dev, "counter reset/update failed\n");

  netif_start_queue(dev);

  rtl_unlock_work(tp);

  tp->saved_wolopts = 0;
  pm_runtime_put_noidle(&pdev->dev);

  rtl8169_check_link_status(dev, tp, ioaddr);
out:
  return retval;

err_release_fw_2:
  rtl_release_firmware(tp);
  rtl8169_rx_clear(tp);
err_free_rx_1:
  dma_free_coherent(&pdev->dev, R8169_RX_RING_BYTES, tp->RxDescArray,
                    tp->RxPhyAddr);
  tp->RxDescArray = NULL;
err_free_tx_0:
  dma_free_coherent(&pdev->dev, R8169_TX_RING_BYTES, tp->TxDescArray,
                    tp->TxPhyAddr);
  tp->TxDescArray = NULL;
err_pm_runtime_put:
  pm_runtime_put_noidle(&pdev->dev);
  goto out;
}
```

