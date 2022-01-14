# HTTP服务端
> 服务端示例
```go
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
)

func main() {
	http.HandleFunc("/demo", func(w http.ResponseWriter, r *http.Request) {
		if r.Method == http.MethodGet {
			// 处理GET请求
			paramMap := make(map[string]interface{}, 0)
			paramStr := r.URL.RawQuery
			paramList := strings.Split(strings.TrimSpace(paramStr), "&")
			for _, v := range paramList {
				param := strings.Split(v, "=")
				paramMap[param[0]] = param[1]
			}
			j, err := json.Marshal(paramMap)
			if err != nil {
				fmt.Println(err)
				return
			}
			res := fmt.Sprintf(`{"status":"ok","msg":"This Is Get Method.","param":"%s"}`, string(j))
			if _, err = w.Write([]byte(res)); err != nil {
				fmt.Println(err)
				return
			}
		}

		if r.Method == http.MethodPost {
			// 处理POST请求
			data, err := ioutil.ReadAll(r.Body)
			if err != nil {
				fmt.Println(err)
				return
			}
			res := fmt.Sprintf(`{"status":"ok","msg":"This Is Post Method.","data":"%s"}`, string(data))
			if _, err = w.Write([]byte(res)); err != nil {
				fmt.Println(err)
				return
			}
		}

	})
	if err := http.ListenAndServe(":8000", nil); err != nil {
		fmt.Println(err)
		return
	}
}
```
> 自定义server
```
s := &http.Server{
	Addr:           ":8080",
	Handler:        myHandler,
	ReadTimeout:    10 * time.Second,
	WriteTimeout:   10 * time.Second,
	MaxHeaderBytes: 1 << 20,
}
log.Fatal(s.ListenAndServe())
```