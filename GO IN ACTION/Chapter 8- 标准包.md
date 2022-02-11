Go 标准库是一组核心包，用来扩展和增强语言的能力

# 文档与源代码

* 不管用什么方式安装 Go，标准库的源代码都会安装在`$GOROOT/src/pkg`
* 标准库的源代码是经过预编译的。这些预编译后的文件，称作归档文件（archive file），

----
# 记录日志

* `stdout`的设备默认目的地是标准文本输出
* `stderr`的设备被创建为日志的默认目的地。这样开发人员就能将程序的输出和日志(执行细节)分离开来
* 更常用的方式是将一般的日志信息写到`stdout`，将错误或者警告信息写到`stderr`

## log包

```
package main

import (
	"log"
)

func init(){

	log.SetPrefix("TRACE: ")
	log.SetFlags(log.Ldate | log.Lmicrosecons | log.Llongfile)
}

func main(){

	log.Println("message")
	log.Fatalln("fatal message") // 在调用 Println()之后会接着调用 os.Exit(1)
	log.Panicln("panic. message") //在调用 Println()之后会接着调用 panic()

}
```

## 定制的日志记录器

```
package main

import(

    "io"
    "io/ioutil"
    "log"
    "os"

)

var (
    Trace *log.Logger
    infor *log.Logger

)

func init(){

    file,err := os.OpenFile("errors.txt", os.O_CREATE|os.O_WRONLY|os.O_APPEND,666)
    if err!= nil(
        log.Fatall("Failed to open error log file", err)
    )
    Trace = log.New(ioutil.Discard, "TRACE: ", log.Ldate|log.Ltime|log.Lshortfile)
    Info = log.New(os.Stdout,"INFO: ",log.Ldate|log.Ltime|log.Lshortfile)

}

```
* 为了创建每个日志记录器，我们使用了 log 包的 New 函数，它创建并正确初始化一个Logger 类型的值
* log 包的实现，是基于对记录日志这个需求长时间的实践和积累而形成的。将
  * 输出写到`stdout`
  * 将日志记录到 `stderr`

---
# 编码/解码

## 解码 JSON

使用 json 包的`NewDecoder`函数以及`Decode`方法进行解码。

```
package main

import (

    "encoding/json"
    "fmt"
    "log"
    "net/http"
)

func main(){

    url :=""
    resp,err := http.Get(uri)
    if err != nil{
        log.Println("ERROR:", err)
        return
    }
    defer resp.Body.Close()
    var gr gResponse
    err = json.NewDecoder(resp.Body).Decode(&gr)
    if err !=nil{
      log.Println("Error",err)
      return
    }
    fmt.Println(gr)
}
```

* 函数`NewDecoder`返回一个指向`Decoder`类型的指针值
* 在读取 JSON响应的过程中，`Decode`方法会将对应的响应解码为这个类型的值,这意味着用户不需要创建对应的值，Decode 会为用户做这件事情


```
var c Contact
err := json.Unmarshal([]byte(JSON), &c) // 将 JSON 字符串反序列化到变量
```
* `Unmarshal`JSON 文档解码到一个结构类型的值里

## 编码 JSON

用`json`包的`MarshalIndent`函数进行编码
* 以很方便地将 Go 语言的 map 类型的值或者结构类型的值转换为易读格式的 JSON 文档

```
func MarshalIndent(v interface{}, prefix, indent string) ([]byte, error) 


c := make(map[string]interface{})
data, err := json.MarshalIndent(c, "", " ")
```
---
# 输入和输出

## Writer 和 Reader 接口

```
type Writer interface{

    Write(p []byte) (n int, err, error)

}
```
* 如果无法全部写入，那么该方法就一定会返回一个错误。
* 返回的写入字节数可能会小于 byte 切片的长度


```
type Reader interface{

      Read(p []byte)(n int, err, error)
}
```

### 整合并完成工作


