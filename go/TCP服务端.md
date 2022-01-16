# TCP服务端

```go
package main

import (
	"bufio"
	"fmt"
	"net"
)

func process(conn net.Conn) {
	defer conn.Close()
	for {
		reader := bufio.NewReader(conn)
		var buf [128]byte
		n, err := reader.Read(buf[:]) // 读取数据
		if err != nil {
			fmt.Println(err)
			break
		}
		msg := fmt.Sprintf("收到client发来的数据：%s", string(buf[:n]))
		fmt.Println(msg)
		conn.Write([]byte(msg)) // 发送数据给client
	}
}

func main() {
	listen, err := net.Listen("tcp", ":8000")
	if err != nil {
		fmt.Println(err)
		return
	}
	for {
		conn, err := listen.Accept() // 建立连接
		if err != nil {
			fmt.Println(err)
			continue
		}
		go process(conn)
	}
}
```