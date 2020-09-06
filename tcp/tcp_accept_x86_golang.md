### TCP Server Accept

依旧从一段平常使用的代码开始：

```go
for {
  	// listener 为 tcp_server 章节构建的 tcp 服务端
    conn, err := listener.Accept()
    if err != nil {
        if netErr, ok := err.(net.Error); ok && netErr.Temporary() {
            continue
        }
        return
    }
    // do something ...
}
```

```go
/* files: go/src/net/tcpsock.go
 */
// Accept implements the Accept method in the Listener interface; it
// waits for the next call and returns a generic Conn.
func (l *TCPListener) Accept() (Conn, error) {
  // 确认 tcp fd 是否成功创建
  if !l.ok() {
    return nil, syscall.EINVAL
  }
  c, err := l.accept()
  if err != nil {
    return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
  }
  return c, nil
}
/* files: go/src/net/tcpsock_posix.go
 */
func (ln *TCPListener) accept() (*TCPConn, error) {
  // golang netFD accept
  // 往下看会详细分析
  fd, err := ln.fd.accept()
  if err != nil {
    return nil, err
  }
  // newTCPConn 方法主要是涉及 fd 封装以及 TCP_NODELAY 设置
  /* TCP_NODELAY 打开该属性的时候，TCP/IP协议栈上将 Nagle 算法关闭
                 kernel:
                 #define DSO_NODELAY	12    // Turn off nagle
   */
  tc := newTCPConn(fd)
  /*
    // ListenConfig contains options for listening to an address.
    // type ListenConfig struct {
    // * If Control is not nil, it is called after creating the network
    // * connection but before binding it to the operating system.
    //
    // * Network and address parameters passed to Control method are not
    // * necessarily the ones passed to Listen. For example, passing "tcp" to
    // * Listen will cause the Control function to be called with "tcp4" or "tcp6".
    // Control func(network, address string, c syscall.RawConn) error
    // * KeepAlive specifies the keep-alive period for network
    // * connections accepted by this listener.
    // * If zero, keep-alives are enabled if supported by the protocol
    // * and operating system. Network protocols or operating systems
    // * that do not support keep-alives ignore this field.
    // * If negative, keep-alives are disabled.
    // KeepAlive time.Duration
    // }
   */
  // 如果协议及系统支持，启用 keepalive，在下面会详细介绍 keepalive
  if ln.lc.KeepAlive >= 0 {
    setKeepAlive(fd, true)
    ka := ln.lc.KeepAlive
    if ln.lc.KeepAlive == 0 {
      ka = defaultTCPKeepAlive
    }
    setKeepAlivePeriod(fd, ka)
  }
  return tc, nil
}
```

- keepalive

```go
/* file: go/src/net/sockopt_posix.go
 * https://man7.org/linux/man-pages/man7/tcp.7.html 
 * SO_KEEPALIVE: SOCKET FD 属性
                 Keep-alives are sent
              	 only when the SO_KEEPALIVE socket option is enabled
 */
func setKeepAlive(fd *netFD, keepalive bool) error {
  err := fd.pfd.SetsockoptInt(syscall.SOL_SOCKET, syscall.SO_KEEPALIVE, boolint(keepalive))
  runtime.KeepAlive(fd)
  return wrapSyscallError("setsockopt", err)
}
/* file: go/src/internal/pool/sockopt.go
 */
func (fd *FD) SetsockoptInt(level, name, arg int) error {
  // incref 为 golang 自旋锁实现的 CAS 操作，增加 fd 的引用，自旋锁这这边不进一步分析，会在单独的锁章节进行讨论
  if err := fd.incref(); err != nil {
    return err
  }
  defer fd.decref()
  /* Syscall6(SYS_SETSOCKOPT, uintptr(s), uintptr(level), uintptr(name), uintptr(val), uintptr(vallen), 0)
   * SYS_SETSOCKOPT:  socket 选项设置
                      ausyscall --dump | grep SETSOCKOPT -i
   * level: 这边需要设置为 syscall.SOL_SOCKET -- 套接字级别设置
   */
  return syscall.SetsockoptInt(fd.Sysfd, level, name, arg)
}
/* file: go/src/tcpsockopt_unix.go
 */
func setKeepAlivePeriod(fd *netFD, d time.Duration) error {
  // The kernel expects seconds so round to next highest second.
  secs := int(roundDurationUp(d, time.Second))
  /* https://man7.org/linux/man-pages/man7/tcp.7.html 
   * TCP_KEEPINTVL: 覆盖内核 tcp keepalive 参数
                    TCP_KEEPINTVL (since Linux 2.4)
                    The time (in seconds) between individual keepalive probes.
                    This option should not be used in code intended to be
                    portable.
   */
  if err := fd.pfd.SetsockoptInt(syscall.IPPROTO_TCP, syscall.TCP_KEEPINTVL, secs); err != nil {
    return wrapSyscallError("setsockopt", err)
  }
  /* https://man7.org/linux/man-pages/man7/tcp.7.html
   * TCP_KEEPIDLE: 覆盖内核 tcp keepalive 参数
                   TCP_KEEPIDLE (since Linux 2.4)
                   The time (in seconds) the connection needs to remain idle
                   before TCP starts sending keepalive probes, if the socket
                   option SO_KEEPALIVE has been set on this socket.  This option
                   should not be used in code intended to be portable.
   */
  err := fd.pfd.SetsockoptInt(syscall.IPPROTO_TCP, syscall.TCP_KEEPIDLE, secs)
  runtime.KeepAlive(fd)
  return wrapSyscallError("setsockopt", err)
}
```

#### KeepAlive 相关内核参数说明总结

> cat  /proc/sys/net/ipv4/tcp_keepalive_*

- tcp_keepalive_time: KeepAlive 的空闲时长，或者说每次正常发送心跳的周期，默认值为7200s（2小时）

- tcp_keepalive_intvl: KeepAlive 探测包的发送间隔，默认值为75s

- tcp_keepalive_probes: 在 tcp_keepalive_time 之后，没有接收到对方确认，继续发送保活探测包次数，默认值为9（次）

#### 应用层覆盖参数：

- TCP_KEEPCNT 覆盖  tcp_keepalive_probes，默认9（次）
- TCP_KEEPIDLE 覆盖 tcp_keepalive_time，默认7200（秒）
- TCP_KEEPINTVL 覆盖 tcp_keepalive_intvl，默认75（秒）

----

> 上述部分分析的是 accept 后对成功建立的 socket 属性的设置（keepalive，nodely），接下去详细分析下 accept 过程

```go
/* file: go/src/net/fd_unix.go
 */
func (fd *netFD) accept() (netfd *netFD, err error) {
  // golang poll accept
  d, rsa, errcall, err := fd.pfd.Accept()
  if err != nil {
    if errcall != "" {
      err = wrapSyscallError(errcall, err)
    }
    return nil, err
  }
  // golang fd 生成
  if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
    poll.CloseFunc(d)
    return nil, err
  }
  if err = netfd.init(); err != nil {
    netfd.Close()
    return nil, err
  }
  /* https://man7.org/linux/man-pages/man2/getsockname.2.html
   * int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
   			 getsockname() returns the current address to which the socket sockfd
         is bound, in the buffer pointed to by addr.  The addrlen argument
         should be initialized to indicate the amount of space (in bytes)
         pointed to by addr.  On return it contains the actual size of the
         socket address.

         The returned address is truncated if the buffer provided is too
         small; in this case, addrlen will return a value greater than was
         supplied to the call.
   */
  lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
  netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
  return netfd, nil
}
```

```go
// Accept wraps the accept network call.
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
  // fd 读锁，增加 fd 读操作的引用，当 fd 不能被用作读操作的时候会上锁失败
  if err := fd.readLock(); err != nil {
    return -1, nil, "", err
  }
  defer fd.readUnlock()
  // 检查 fd 是否准备好
  if err := fd.pd.prepareRead(fd.isFile); err != nil {
    return -1, nil, "", err
  }
  for {
    s, rsa, errcall, err := accept(fd.Sysfd)
    if err == nil {
      return s, rsa, "", err
    }
    switch err {
      /* EAGAIN: Resource temporarily unavailable
       * 暂不可用的情况下
       */
    case syscall.EAGAIN:
      // 如果支持 poll
      if fd.pd.pollable() {
        // waitRead 追踪下去会进行 runtime_pollWait 调用
        if err = fd.pd.waitRead(fd.isFile); err == nil {
          continue
        }
      }
    case syscall.ECONNABORTED:
      // This means that a socket on the listen
      // queue was closed before we Accept()ed it;
      // it's a silly error, so try again.
      continue
    }
    return -1, nil, errcall, err
  }
}
```

```go
/* file: go/src/internal/pool/fd_poll_runtime.go
 */
func runtime_pollWait(ctx uintptr, mode int) int
/* file: go/src/runtime/netpoll.go
 * 这边使用了 //go:linkname 「function name」 「package/func 」
 	 第一个参数表示当前方法或变量
 	 第二个参数表示目标方法或变量，因为这关指令会破坏系统和包的模块化，因此在使用时必须导入 unsafe // import "unsafe"
 */
//go:linkname poll_runtime_pollWait internal/poll.runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
  // Network poller descriptor 异常确认
  err := netpollcheckerr(pd, int32(mode))
  if err != 0 {
    return err
  }
  // As for now only Solaris, illumos, and AIX use level-triggered IO.
  if GOOS == "solaris" || GOOS == "illumos" || GOOS == "aix" {
    netpollarm(pd, mode)
  }
  // go poll 阻塞，直到有连接建立后再进行协程调度
  for !netpollblock(pd, int32(mode), false) {
    err = netpollcheckerr(pd, int32(mode))
    if err != 0 {
      return err
    }
    // Can happen if timeout has fired and unblocked us,
    // but before we had a chance to run, timeout has been reset.
    // Pretend it has not happened and retry.
  }
  return 0
}
```

```go
/* file go/src/runtime/netpoll.go
 * 这边粗略看一下 netpollblock 具体干了哪些工作，详细分析会结合协程调度在 goroutine 章节具体分析
 */
// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
  gpp := &pd.rg
  if mode == 'w' {
    gpp = &pd.wg
  }
  
  // set the gpp semaphore to WAIT
  for {
    old := *gpp
    if old == pdReady {
      *gpp = 0
      return true
    }
    if old != 0 {
      throw("runtime: double wait")
    }
    // cas 比较赋值操作
    if atomic.Casuintptr(gpp, 0, pdWait) {
      break
    }
  }
  
  
  // need to recheck error states after setting gpp to WAIT
  // this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
  // do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
  if waitio || netpollcheckerr(pd, mode) == 0 {
    // gopark 函数：协程标记等待状态，挂起等待 runtime 调度
    gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
  }
  // be careful to not lose concurrent READY notification
  old := atomic.Xchguintptr(gpp, 0)
  if old > pdWait {
    throw("runtime: corrupted polldesc")
  }
  return old == pdReady
}
```

```go
/* file: go/src/internal/pool/sock_cloexec.go
 *  s, rsa, errcall, err := accept(fd.Sysfd)
 *  fd.Sysfd 服务端监听 socket 套接字，具体生成过程可在 tcp_server_x86_golang.md 中查看
 */
func accept(s int) (int, syscall.Sockaddr, string, error) {
  ns, sa, err := Accept4Func(s, syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
  // On Linux the accept4 system call was introduced in 2.6.28
  // kernel and on FreeBSD it was introduced in 10 kernel. If we
  // get an ENOSYS error on both Linux and FreeBSD, or EINVAL
  // error on Linux, fall back to using accept.
  switch err {
  case nil:
    return ns, sa, "", nil
  default: // errors other than the ones listed
    return -1, sa, "accept4", err
  case syscall.ENOSYS: // syscall missing
  case syscall.EINVAL: // some Linux use this instead of ENOSYS
  case syscall.EACCES: // some Linux use this instead of ENOSYS
  case syscall.EFAULT: // some Linux use this instead of ENOSYS
  }

	// See ../syscall/exec_unix.go for description of ForkLock.
	// It is probably okay to hold the lock across syscall.Accept
	// because we have put fd.sysfd into non-blocking mode.
	// However, a call to the File method will put it back into
	// blocking mode. We can't take that risk, so no use of ForkLock here.
  
  /* var Accept4Func func(int, int) (int, syscall.Sockaddr, error) = syscall.Accept4
   * accept 在 kernel 中的过程：
         tcp connect => NET => server => syn queue => accept queue
         调用 accept 从 accept queue 中取出成功建立的 tcp
         socket 在 accpet 过程中会进行相应地址（端口）的分配，端口分配范围为内核参数 ip_local_port_range 决定 
         cat /proc/sys/net/ipv4/ip_local_port_range
         对应 kernel 操作：
     /* Update system visible IP port range */
     static void set_local_port_range(struct net *net, int range[2])
     {
         bool same_parity = !((range[0] ^ range[1]) & 1);

         write_seqlock_bh(&net->ipv4.ip_local_ports.lock);
         if (same_parity && !net->ipv4.ip_local_ports.warned) {
            net->ipv4.ip_local_ports.warned = true;
            pr_err_ratelimited("ip_local_port_range: prefer different parity for start/end values.\n");
         }
         net->ipv4.ip_local_ports.range[0] = range[0];
         net->ipv4.ip_local_ports.range[1] = range[1];
         write_sequnlock_bh(&net->ipv4.ip_local_ports.lock);
    }
   */
  
  ns, sa, err = AcceptFunc(s)
  if err == nil {
    // CloseOnExec 设置，具体可在 tcp_server_x86_golang 中查看
    syscall.CloseOnExec(ns)
  }
  if err != nil {
    return -1, nil, "accept", err
  }
  // IO 非阻塞设置
  if err = syscall.SetNonblock(ns, true); err != nil {
    CloseFunc(ns)
    return -1, nil, "setnonblock", err
  }
  return ns, sa, "", nil
}
```

