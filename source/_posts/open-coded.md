---
title: Go开放编码
date: 2025-03-31 23:12:09
tags:
  - Go
  - 开放编码
  - 面试
categories: Go
---

# 问题发现
在一次面试中的笔试题里，有这样一个问题：
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

问：输出结果是什么？

我的理解输出应该是`error,3,3,3,John,`
- 首先`defer`是延迟调用，先进后出的；
- 调用`panic`，之前的`defer`会被执行，后面的`defer`不会被执行；
- `panic`被`recover`捕获，输出panic的信息`error,`；
- `for`循环中的`defer`是一个闭包，输出的是j的最后的值, 也就是`3,3,3,`；
- 最后的name传参了, 复制了一份, `name = "Lee"`不会影响；
- 最终输出`error,3,3,3,John,`

在Goland中直接运行以后，结果确实是`error,3,3,3,John,`，但是在朋友的电脑上运行，结果是`error,2,1,0,John,`，这才发现了不一致的地方。

之后我在Go Playground中运行, 同时直接使用`go run`命令运行，结果都是`error,2,1,0,John,`。
# 原因分析
Go 1.14+版本启用开放编码优化。
- 编译器将循环中的`defer`优化为每次迭代独立注册，捕获当前迭代的变量j的值。
  相当于每次生成一个副本(`jTmp := j`)
- 通过延长比特(defer bit)机制记录每次迭代的独立状态。

开放编码优化的触发条件:
- 函数内`defer`总数不超过8个；
- `defer`不在循环内部（但实际优化可能覆盖部分循环场景）；
- 未禁用编译器优化。