# UDP客户端

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"strings"
)

func main() {
	// 建立连接
	socket, err := net.DialUDP("udp", nil, &net.UDPAddr{
		IP:   net.IPv4(0, 0, 0, 0),
		Port: 3000,
	})
	if err != nil {
		fmt.Println(err)
		return
	}
	defer socket.Close()
	for {
		inputReader := bufio.NewReader(os.Stdin) // 接收输入
		input, err := inputReader.ReadString('\n')
		if err != nil {
			fmt.Println(err)
			continue
		}
		inputInfo := strings.Trim(input, "\r\n")
		if strings.ToUpper(inputInfo) == "Q" {
			return
		}
		socket.Write([]byte(inputInfo)) // 发送数据
		var buf [1024]byte
		n, _, err := socket.ReadFromUDP(buf[:]) // 接收数据
		msg := fmt.Sprintf("来自服务端:%s", string(buf[:n]))
		fmt.Println(msg)
	}
}

```