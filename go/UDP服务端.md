# UDP服务端
```go
package main

import (
	"fmt"
	"net"
)

func main() {
	listen, err := net.ListenUDP("udp", &net.UDPAddr{
		IP:   net.IPv4(0, 0, 0, 0),
		Port: 3000,
	})
	if err != nil {
		fmt.Println(err)
		return
	}
	defer listen.Close()
	for {
		var data [1024]byte
		n, addr, err := listen.ReadFromUDP(data[:]) // 接收数据
		if err != nil {
			fmt.Println(err)
			continue
		}
		msg := fmt.Sprintf("收到%s:%s", addr, string(data[:n]))
		fmt.Println(msg)
		_, err = listen.WriteToUDP([]byte(msg), addr) // 发送数据
		if err != nil {
			fmt.Println(err)
			continue
		}
	}
}

```