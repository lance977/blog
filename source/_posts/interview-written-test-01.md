---
title: 面试笔试题-01
date: 2025-04-01 11:08:24
tags:
    - Go
    - 面试
    - 笔试
categories: Go
---
**1. 请简述Golang有哪些方法可以实现调用C/C++代码（包括非标准的方法）？**

**2. 请写一段简短的代码实现C语言中的char\*转换成Golang中的[]byte？**

**3. 请您简短介绍下反射具体可以实现哪些功能，如果您在工作中曾经使用过反射，请说说具体的应用场景**

**4. 阅读以下源码，请写出程序运行结果**

```go
package main

import "fmt"

func main() {
	var peoples map[string]string = make(map[string]string)
	peoples["001"] = "Jack"
	peoples["004"] = "John"
	peoples["003"] = "Georgia"
	peoples["002"] = "Lucy"
	for k, v := range peoples {
		fmt.Println(k, v)
	}
}
```
输出结果：
golang中map的遍历是无序的，每次遍历的顺序可能不同。
```
001 Jack
004 John
003 Georgia
002 Lucy
```

**5. 阅读以下源码，如果编译报错，答案请填写“编译报错”四个字，如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）？**

```go
package main

import (
	"encoding/json"
	"fmt"
)

type People struct {
	name string
	age int
}

var data string = `{
						"name": "Jane",
						"age": 22,
						"Name": "Ella",
						"Age": 20
					}`

func main() {
	var people People
	json.Unmarshal([]byte(data), &people)
	fmt.Printf("%s,%d",people.name,people.age)
}
```
输出结果：
```
,0
```
解析：结构体`People`的字段`name`和`age`是未导出的，通过`json.Unmarshal`解析JSON数据时，会忽略这些字段。

**6. 阅读以下源码，如果编译报错，答案请填写“编译报错”四个字，如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）（运行在64位系统下）？**

```go
package main

import (
	"fmt"
	"unsafe"
)

type Item struct {
	books []string
}

func main() {
	d := Item{}
	fmt.Printf("%d", unsafe.Sizeof(d.books))
}
```
输出结果：
```
24
```
解析：在64位系统下，指针类型的大小是8字节，切片类型的大小是24字节。
切片类型包含三个字段：指向底层数组的指针、切片的长度和切片的容量。因此，切片类型的大小是8字节（指针类型的大小）+8字节（切片的长度）+8字节（切片的容量）=24字节。

**7. 阅读以下源码，如果编译报错，答案请填写“编译报错”四个字，如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）？**

```go
package main

import (
	"fmt"
	"time"
)

var receive = make(chan int, 1)
var wait = make(chan bool)

func main() {
	go func() {
		i := 0
		for i < 3 {
			receive <- i
			fmt.Printf("%d,", i)
			i++
		}
	}()
	go func() {
		for {
			select {
			case v := <-receive:
				time.Sleep(1 * time.Second)
				fmt.Printf("%d,", v)
				if v == 2 {
					wait <- true
				}
			}
		}
	}()
	<-wait
	fmt.Printf("10,")
	time.Sleep(60 * time.Second)
}
```
输出结果：
```
0,1,0,2,1,2,10,
```
解析：`receive`是缓冲区大小为1的`channel`，`wait`是无缓冲区的`channel`。该题主要问题是第二个协程sleep 1秒然后打印, 在这个过程中, 第一个协程会往`receive`中写入下一个数据并打印, 因此会在第二个协程打印之前。
- 第一个协程会往`receive`中写入0,1,2, 并打印写入的值，然后退出。
- 第二个协程会从`receive`中读取值，sleep 1秒，再打印读取的值，然后判断是否为2，如果是2，则往`wait`中写入`true`，然后退出。
- 主协程会阻塞，直到从`wait`中读取值，打印`10,`，然后sleep 60秒。
- 首先第一个协程向`receive`中写入0, 然后打印出`0,`，因为缓冲区已满，协程会阻塞；
- 第二个协程从`receive`中读取到0, sleep 1秒，然后打印出`0,`;
- 在第二个协程从sleep到打印这段时间，`receive`数据已经被读取了，所以第一个协程继续写入`1`并打印出`1,`, 然后再次阻塞，等待读取后再写入，因此1会在第二个协程打印出`0,`之前打印；
- 直到第二个协程从`receive`中读取到2，sleep 1秒，然后打印出`2,`，然后往`wait`中写入`true`，然后退出；
- 主协程从`wait`中读取到`true`，打印出`10,`，然后sleep 60秒。
- 因此最终结果包含顺序是`0(一),1(一),0(二),2(一),1(二),2(二),10(主),`。

**8. 阅读以下源码，如果编译报错，答案请填写“编译报错”四个字，如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）？**

```go
package main

import (
	"fmt"
)

func main() {
	var object interface{}
	object = "string"
	object = 1
	switch object {
	case 1:
		fmt.Printf("1")
		fallthrough
	case "string":
		fmt.Printf("2")
	case 2:
		fmt.Printf("3")
		break
	case 3:
		fmt.Printf("4")
	default:
		fmt.Printf("5 ")
	}
}
```
输出结果：
```
12
```
解析: 
- `object`是一个`interface{}`类型，因此可以赋值为任意类型。
- 先赋值为`"string"`，然后赋值为`1`，因此`object`的值是`1`。
- `switch`语句匹配到`case 1`，打印`1`。
- 遇到`fallthrough`语句，会继续执行下一个`case`，打印`2`。
- go语言的`swithc-case`与其他语言不同的是, 它会自动break, 因此不会执行后面的`case`。


**9. 阅读以下源码，如果编译报错，答案请填写“编译报错”四个字，如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）？**

```go
package main

import "fmt"

func main() {
	Defer("John")
}

func Defer(name string) {
	defer func(param string) {
		fmt.Printf("%s,", param)
	}(name)
	for j := 0; j < 3; j++ {
		defer func() {
			fmt.Printf("%d,", j)
		}()
	}
	defer func() {
		err := recover()
		if err != nil {
			fmt.Printf("%s,", err)
		}
	}()
	name = "Lee"
	panic("error")
	defer func() {
		fmt.Printf("end,")
	}()
}
```
输出结果：
```
error,2,1,0,John,
```
解析：
- 首先`defer`是延迟调用，先进后出的；
- 调用`panic`，之前的`defer`会被执行，后面的`defer`不会被执行；
- `panic`被`recover`捕获，输出panic的信息`error,`；
- `for`循环中的`defer`启用开放编码优化，输出`2,1,0`；
  ```go
    for j := 0; j < 3; j++ {
        temp := j // 启用开放编码优化, 相当于增加了这样一个字段
        defer func() { fmt.Printf("%d,", temp) }()
    }
  ```
- 最后的name传参了, 复制了一份, `name = "Lee"`不会影响；
- 最终输出`error,3,3,3,John,`

**10. 阅读以下源码，如果编译报错，答案请填写“编译报错”四个字如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）？**

```go
package main

import (
	"fmt"
)

type People struct {
	age  *int
	name string
}

func NewPeople(name string, age int) (p *People) {
	p = new(People)
	p.age = new(int)
	p.SetName(name)
	p.SetAge(age)
	return
}

func (p People) SetAge(age int) {
	p.age = &age
}

func (p People) GetAge() int {
	return *p.age
}

func (p People) SetName(name string) {
	p.name = name
}

func (p People) GetName() string {
	return p.name
}

func main() {
	var people *People = NewPeople("John", 22)
	people.SetName("Grace")
	people.SetAge(45)
	fmt.Printf("%s,%d", people.GetName(), people.GetAge())
}
```
输出结果：
```
,0 
```
解析: 主要原因是方法接收器类型是**值类型**，因此修改方法接收器的值不会影响到原始对象。如果是指针类型`func (p *People) SetAge(age int) {}`就可以改变原始对象的值。

**11. 阅读以下源码如果编译报错，答案请填写“编译报错”四个字，如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）？**

```go
package main

import (
	"fmt"
)

var item = "hello"

func main() {
	v := item
	v[0] = 'a'
	fmt.Printf("%s", item)
}
```
输出结果：编译器报错。string是不可变的，不能修改。

**12. 阅读以下源码，如果编译报错，答案请填写“编译报错”四个字，如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）？**

```go
package main

import (
	"fmt"
)

type Callback interface {
	OnProgress(progress int)
}

type Download struct {
}

func (d *Download) OnProgress(progress int) {
	fmt.Println(progress)
}

func GetDownload(role string) Callback {
	var d *Download
	if role == "admin" {
		d = new(Download)
	}
	return d
}

func main() {
	c := GetDownload("visitor")
	if c != nil {
		c.OnProgress(100)
	} else {
		c.OnProgress(0)
	}
}
```
输出结果：
```
100
```
解析: 主要考查接口类型的nil值，`interface{}`包含了类型和值，只有类型和值都为nil时，接口类型才为nil。
- `Download`实现了`Callback`接口；
- `GetDownload`返回的实际是`*Download`类型；
- 此时`c`的类型`*Download`，值为`nil`，因此`c`不为nil，会调用`c.OnProgress(100)`，输出`100`；

**13. 阅读以下源码，如果编译报错答案请填写“编译报错”四个字，如果有运行结果，答案请填写运行结果（严格区分大小写、请勿写多余的空格）？**

````go
package main

import "fmt"

func TestSlice(arr []int) {
	arr = append(arr, 3)
	arr = append(arr, 4)
}

func main() {
	arr := make([]int, 1, 10)
	arr = append(arr, 1)
	arr = append(arr, 2)
	TestSlice(arr)
	fmt.Printf("%d,%d,%d", arr[0], arr[1], len(arr))
}
````
输出结果：
```
0,1,3
```
解析:
- `arr := make([]int, 1, 10)`创建了一个长度为1，容量为10的切片, 此时arr的值为`[0]`；
- `arr = append(arr, 1)`和`arr = append(arr, 2)`追加了两个元素，此时arr的值为`[0, 1, 2]`；
- Go语言中，切片是引用类型，但函数传参是按值传递的，因此`TestSlice`函数中的`arr`是一个新的切片，指向的是同一个底层数组，因此在`TestSlice`函数中修改`arr`的值，不会影响到`main`函数中的`arr`的值；
- 最终输出`0,1,3`。
