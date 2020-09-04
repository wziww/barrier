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
  return syscall.SetsockoptInt(fd.Sysfd, level, name, arg)
}
```

