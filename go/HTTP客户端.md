# HTTP客户端
   
> GET请求示例
```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {

	// GET请求
	resp, err := http.Get("http://www.keymak1r.com")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(body))
}
```
> 带参数GET请求示例
```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"net/url"
)

func main() {

	// 带参数GET请求
	path, err := url.ParseRequestURI("http://www.keymak1r.com")
	if err != nil {
		fmt.Println(err)
		return
	}
	param := url.Values{}
	param.Set("name", "wbw")
	path.RawQuery = param.Encode()
	fmt.Println(path.String())
	resp, err := http.Get(path.String())
	if err != nil {
		fmt.Println(err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(body))
}
```
> POST请求示例
```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
)

func main() {

	// POST请求
	data := `{"name":"wbw"}`
	resp, err := http.Post("http://www.keymak1r.com", "application/json", strings.NewReader(data))
	if err != nil {
		fmt.Println(err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(body))
}
```

>自定义 HTTP Client

要管理HTTP客户端的头域、重定向策略和其他设置，创建一个Client：
```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
)

func main() {

	// 自定义Client
	client := http.Client{}
	// 创建request
	data := `{"name":"wbw"}`
	req, err := http.NewRequest(http.MethodPost, "http://www.keymak1r.com", strings.NewReader(data))
	if err != nil {
		fmt.Println(err)
		return
	}
	// 设置请求头
	req.Header.Set("Content-Type", "application/json")
	// 发起请求
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(body))
}
```
> 自定义Transport

要管理代理、TLS配置、keep-alive、压缩和其他设置，创建一个Transport：
```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
)

func main() {

	// 自定义Transport
	tr := &http.Transport{
		DisableCompression: true,
	}
	// 自定义client
	client := http.Client{
		Transport: tr,
	}
	// 创建request
	data := `{"name":"wbw"}`
	req, err := http.NewRequest(http.MethodPost, "http://www.keymak1r.com", strings.NewReader(data))
	if err != nil {
		fmt.Println(err)
		return
	}
	// 设置请求头
	req.Header.Set("Content-Type", "application/json")
	// 发起请求
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(body))
}
```
**Client和Transport类型都可以安全的被多个goroutine同时使用。出于效率考虑，应该一次建立、尽量重用。**