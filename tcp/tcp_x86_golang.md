### 从 golang api 说起

```golang
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

```golang

/* 
 * 例：internetSocket(ctx, "tcp", localAddr, nil, syscall.SOCK_STREAM, 0, "listen", sl.ListenConfig.Control) 
 * net   							  网络类别
 * laddr  local address  本地地址
 * raddr  remote address 远程地址，这边的例子是创建本地 tcp 服务器，所以远程地址不需要填
 * sotype socket type   socket 类型，tcp 使用 syscall.SOCK_STREAM
     |
     *— syscall.SOCK_STREAM     TCP
     *— syscall.SOCK_DGRAM      UDP
 */

func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
	if (runtime.GOOS == "aix" || runtime.GOOS == "windows" || runtime.GOOS == "openbsd") && mode == "dial" && raddr.isWildcard() {
		raddr = raddr.toLocal(net)
	}
	family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
	return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr, ctrlFn)
}
```



