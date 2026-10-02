
```go
package main
import "fmt"
func main() {
    fmt.Println("Google" + "Runoob")
}
```

Go 语言中使用 fmt.Sprintf 或 fmt.Printf 格式化字符串并赋值给新串：

- **Sprintf** 根据格式化参数生成格式化的字符串并返回该字符串。
- **Printf** 根据格式化参数生成格式化的字符串并写入标准输出。

## Sprintf Printf 实例


```go
package main  
import (  
    "fmt"  
)  
func main() {  
   // %d 表示整型数字，%s 表示字符串  
    var stockcode=123  
    var enddate="2020-12-31"  
    var url="Code=%d&endDate=%s"  
    
    var target_url=fmt.Sprintf(url,stockcode,enddate)  
    fmt.Println(target_url)  
}  


输出结果为：
Code=123&endDate=2020-12-31


package main  
  
import (  
    "fmt"  
)  
  
func main() {  
   // %d 表示整型数字，%s 表示字符串  
    var stockcode=123  
    var enddate="2020-12-31"  
    var url="Code=%d&endDate=%s"  
    fmt.Printf(url,stockcode,enddate)  
}

输出结果为：
Code=123&endDate=2020-12-31
```
转成2进制字符串：`fmt.Sprintf("%b", n)`
```go
a := 10          // a 是 int
b := 3.14        // b 是 float64
c := "hello"     // c 是 string
d := true        // d 是 bool
e := []int{1,2}  // e 是 []int
a, b := 2, 3  // a 是赋值，b 是新声明，合法
```

## 声明数组的几种写法
