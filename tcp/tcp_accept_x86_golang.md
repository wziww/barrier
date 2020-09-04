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

