## golang TCP 例子

### C 风格 tcp 相关操作

```go
package main

import (
	"syscall"
)

func main() {
  /*
   * syscall.AF_INET        IPv4
   * syscall.SOCK_STREAM    TCP
   */
	syscall.ForkLock.RLock()
  //
	fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0)
	if err != nil {
		panic(err)
	}
  /*
   * syscall.SetNonblock(fd, true) 等同于： 
   * flag, _, errno := syscall.Syscall(syscall.SYS_FCNTL, uintptr(fd), uintptr(syscall.F_GETFL), 0)
	 * flag |= syscall.O_NONBLOCK
	 * _, _, errno = syscall.Syscall(syscall.SYS_FCNTL, uintptr(fd), uintptr(syscall.F_SETFL), flag)
	 */
	err = syscall.SetNonblock(fd, true)
	if err != nil {
		panic(err)
	}
	syscall.CloseOnExec(fd)
	syscall.ForkLock.RUnlock()
}
```

