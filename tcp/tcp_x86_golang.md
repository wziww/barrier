## 1.14.3

### 从 golang API 说起

```go
	// 创建一个 tcp 服务
	listener, err := net.Listen("tcp", address)
	if err != nil {
		panic(err)
	}
	for {
		conn, err := listener.Accept()
		do something
	}
```

此处使用 golang 标准库 net 创建了一个 tcp 服务并进行监听，首先来主要分析下
net.Listen 干了些什么

```golang
// go/src/net/tcpsock_posix.go
func (sl *sysListener) listenTCP(ctx context.Context, laddr *TCPAddr) (*TCPListener, error) {
	fd, err := internetSocket(ctx, sl.network, laddr, nil, syscall.SOCK_STREAM, 0, "listen", sl.ListenConfig.Control)
	if err != nil {
		return nil, err
	}
	return &TCPListener{fd: fd, lc: sl.ListenConfig}, nil
}
```

internetSocket 主要参数说明：

```go

/* file: go/src/net/ipsock_posix.go
 * 例：internetSocket(ctx, "tcp", localAddr, nil, syscall.SOCK_STREAM, 0, "listen", sl.ListenConfig.Control) 
 * net   							  网络类别
 * laddr  local address  本地地址
 * raddr  remote address 远程地址，这边的例子是创建本地 tcp 服务器，所以远程地址不需要填
 * sotype socket type   socket 类型，tcp 使用 syscall.SOCK_STREAM
     |
     *— syscall.SOCK_STREAM     TCP
		 *— syscall.SOCK_DGRAM      UDP
 * proto
 * mode  模式，此处监听用 listen
 */

func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
	if (runtime.GOOS == "aix" || runtime.GOOS == "windows" || runtime.GOOS == "openbsd") && mode == "dial" && raddr.isWildcard() {
		raddr = raddr.toLocal(net)
	}
	family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
	return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr, ctrlFn)
}
```
favoriteAddrFamily 函数说明

```go
/* file: go/src/net/ipsock_posix.go
 * 套接字参数说明：
 		|
 		*- syscall.AF_INET  IPv4
 		*- syscall.AF_INET6 IPv6 和 IPv4
 */
func favoriteAddrFamily(network string, laddr, raddr sockaddr, mode string) (family int, ipv6only bool) {
	switch network[len(network)-1] {
	case '4':
		return syscall.AF_INET, false
	case '6':
		return syscall.AF_INET6, true
	}
	// local addr 参数以及系统 ip 支持程度校验
	if mode == "listen" && (laddr == nil || laddr.isWildcard()) {
		if supportsIPv4map() || !supportsIPv4() {
			return syscall.AF_INET6, false
		}
		if laddr == nil {
			return syscall.AF_INET, false
		}
		return laddr.family(), false
	}

	if (laddr == nil || laddr.family() == syscall.AF_INET) &&
		(raddr == nil || raddr.family() == syscall.AF_INET) {
		return syscall.AF_INET, false
	}
	return syscall.AF_INET6, false
}
```

---

### Socket

```go
/* file: go/src/net/sock_posix.go
 * tcp 创建总函数，里面分为多个步骤，在下面的分析会逐一说明
 */ 
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
  // 系统调用进行指定类型（这边以 tcp socket 为例）套接字创建
	s, err := sysSocket(family, sotype, proto)
	if err != nil {
		return nil, err
	}
  // 系统调用套接字基础属性设置
	if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}
  // 根据上面创建的套接字生成 golang netfd
	if fd, err = newFD(s, family, sotype, net); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}
	if laddr != nil && raddr == nil {
		switch sotype {
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
      // 套接字监听属性设置以及本地端口绑定
			if err := fd.listenStream(laddr, listenerBacklog(), ctrlFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		case syscall.SOCK_DGRAM:
			if err := fd.listenDatagram(laddr, ctrlFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
	}
	if err := fd.dial(ctx, laddr, raddr, ctrlFn); err != nil {
		fd.Close()
		return nil, err
	}
	return fd, nil
}
```

### sysSocket(family, sotype, proto)

```go
/* file: go/src/net/sock_cloexec.go
*/
func sysSocket(family, sotype, proto int) (int, error) {
  /* 系统调用生成 sock 套接字
   * 参数说明：
   * syscall.SOCK_NONBLOCK fd 非阻塞设置
   * syscall.SOCK_CLOEXEC  (since Linux 2.6.23)
         #define SOCK_CLOEXEC	O_CLOEXEC // linux kernel
         https://man7.org/linux/man-pages/man2/open.2.html
         这边是在进行系统调用 open 时设置了 fd 的 O_CLOEXEC 属性，这是个原子性的操作
         也可以在打开文件后 fcntl F_SETFD 设置 fd 属性，但是会有并发风险
         此时打开文件和设置属性分为了两个步骤，可能会造成在间隙期间依旧出现不良后果的情况
         该属性具体解释：
         O_CLOEXEC 是为了在多线程场景下避免 fd 泄露给子线程造成一些诸如子线程非法使用父线程打开的文件，子线程非法持有父线程 				 socket，当父线程退出的时候子线程依旧持有 socket fd 从而导致例如端口依旧被占用之类的问题 
         
   */
	s, err := socketFunc(family, sotype|syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC, proto)
	switch err {
	case nil:
		return s, nil
	default:
		return -1, os.NewSyscallError("socket", err)
	case syscall.EPROTONOSUPPORT, syscall.EINVAL:
	}
	syscall.ForkLock.RLock()
	s, err = socketFunc(family, sotype, proto)
	if err == nil {
		syscall.CloseOnExec(s)
	}
	syscall.ForkLock.RUnlock()
	if err != nil {
		return -1, os.NewSyscallError("socket", err)
	}
  // 非阻塞设置
	if err = syscall.SetNonblock(s, true); err != nil {
		poll.CloseFunc(s)
		return -1, os.NewSyscallError("setnonblock", err)
	}
	return s, nil
}
```





