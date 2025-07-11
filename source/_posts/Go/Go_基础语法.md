---
title: Go_基础语法
typora-root-url: ../..
categories:
  - [Go]
tags:
  - null 
date: 2023/5/18
update:
comments:
published:
---

# Work before Hello World

1. Open a command prompt and cd to home directory.

2. Create a hello directory for the first Go source code.

   ```bash
   mkdir hello
   cd hello
   ```

3. Enable dependency tracking for your code.

   When your code imports packages contained in other modules, you manage those dependencies through your code's own module. **That module is defined by a `go.mod` file that tracks the modules that provide those packages**. That go.mod file stays with your code, including in your source code repository.

   To enable dependency tracking for your code by creating a go.mod file, run the [`go mod init` command](https://go.dev/ref/mod#go-mod-init), giving it the name of the module your code will be in. The name is the module's module path.
   In actual development, the module path will typically be the repository location where your source code will be kept. For example, the module path might be `github.com/mymodule`. If you plan to publish your module for others to use, the module path *must* be a location from which Go tools can download your module. For more about naming a module with a module path, see [Managing dependencies](https://go.dev/doc/modules/managing-dependencies#naming_module).
   But for the purposes of just a tutorial, just use `example/hello`.

   ```bash
   $ go mod init example/hello
   go: creating new go.mod: module example/hello
   ```

4. In your text editor, create a file hello.go in which to write your code.

5. Paste the following code into your hello.go file and save the file.
   ```go
   package main
   
   import "fmt"
   
   func main(){
       fmt.Println("Hello, World!")
   }
   ```

   This is your Go code. In this code, you:

   * Declare a `main` **package (a package is a way to group functions, and it's made up of all the files in the same directory)**.
   * Import the popular [`fmt` package](https://pkg.go.dev/fmt/), which contains functions for formatting text, including printing to the console. This package is one of the [standard library](https://pkg.go.dev/std) packages you got when you installed Go.
   * Implement a `main` function to print a message to the console. A `main` function executes by default when you run the `main` package.

6. Run the code to see the greeting.

   ```bash
   $ go run .
   Hello, World!
   ```

## Key Points

1. When your code imports packages contained in other modules, you manage those dependencies through your code's own module. **That module is defined by a `go.mod` file that tracks the modules that provide those packages**. That go.mod file stays with your code, including in your source code repository.
2. To enable dependency tracking for your code by creating a go.mod file, run the [`go mod init` command](https://go.dev/ref/mod#go-mod-init), giving it the name of the module your code will be in. The name is the module's module path.
3. In actual development, the module path will typically be the repository location where your source code will be kept. For example, the module path might be `github.com/mymodule`. If you plan to publish your module for others to use, the module path *must* be a location from which Go tools can download your module. For more about naming a module with a module path, see [Managing dependencies](https://go.dev/doc/modules/managing-dependencies#naming_module).
4. Package is a way to group functions, and it's made up of all the files in the same directory.
5. A `main` function executes by default when you run the `main` package.
6. go mod init command: `go mod init example/hello `
7. go run command: `go run example/hello`
8. go build command and run: `go build example/hello/main.go` `./example/hello/main`

# variable type

```go
package main

import (
	"fmt"

	"math"
)

func main() {
	var a = "initial"
	var b, c int = 1, 2
	var d = true
	var e float64
	f := float32(e)
	g := a + "foo"
	fmt.Println(a, b, c, d, e, f)
	fmt.Println(g)

	const s string = "constant"
	const h = 500000000
	const i = 3e20 / h
	fmt.Println(s, h, i, math.Sin(h), math.Sin(i))
}
```

## Key Points

1. go语言是一门强类型语言，每一个变量都有它自己的变量类型
2. go语言的字符串是内置类型，可以直接通过加号拼接，也能够直接用等于号去比较两个字符串。
3. go语言变量的声明有两种方式
   1. 一种是通过`var name string = ""`这种方式来声明变量，声明变量的时候，一般会自动去推导变量的类型。
   2. 如果有需要，也可以显式写出变量类型。另一种声明变量的方式是：`使用变量 冒号 := 值`。
4. 常量就是把var改成const，值得一提的是go语言里面的常量没有确定的类型，会根据使用的上下文来自动确定类型。

# if else

```go
package main

import "fmt"

func main() {
	if 7%2 == 0 {
		fmt.Println("7 is even")
	} else {
		fmt.Println("7 is odd")
	}
	if 8%4 == 0 {
		fmt.Println("8 is divisible by 4")
	}
	if num := 9; num < 0 {
		fmt.Println(num, "is negative")
	} else if num < 10 {
		fmt.Println(num, "has 1 digit")
	} else {
		fmt.Println(num, "has multiple digits")
	}
}
```

```
7 is odd
8 is divisible by 4
9 has 1 digit
```

## Key Points

1. if后面没有括号。
2. if执行语句块必须接大括号。不能像C或者Cpp一样，直接把if里面的语句同一行。

# for

```go
package main

import "fmt"

func main() {
	i := 1
	for {
		fmt.Println("loop")
		break
	}
	for j := 7; j < 9; j++ {
		fmt.Println(j)
	}
	for n := 0; n < 5; n++ {
		if n%2 == 0 {
			continue
		}
		fmt.Println(n)
	}
	for i <= 3 {
		fmt.Println(i)
		i = i + 1
	}
}
```

```
loop
7
8
1
3
1
2
3
```

## Key Points

1. go语言只有for循环
2. 最简单的for后面什么也不写，是死循环
3. for i等于0，i小于n，i加加。这中间三段，任何一段都可以省略。

# switch

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	a := 2
	switch a {
	case 1:
		fmt.Println("one")
	case 2:
		fmt.Println("two")
	case 3:
		fmt.Println("Three")
	case 4, 5:
		fmt.Println("four or five")
	default:
		fmt.Println("Other")
	}

	t := time.Now()
	switch {
	case t.Hour() < 12:
		fmt.Println("It's before noon")
	default:
		fmt.Println("It's after noon")
	}
}
```

```
two
It's after noon
```

## Key Points

1. go语言的switch后面的那个变量名，不要括号。
2. 很大的一点不同的是，在cpp里面，switch case如果不加break的话会然后会继续往下跑完所有的case，在go语言里面的话不加break也会跳出来。
3. 相比C或者Cpp，go语言里面的switch功能更强大。可以使用任意的变量类型，甚至可以用来取代任意的if else语句。你可以在switch后面不加任何的变量，然后在case里面写条件分支。这样代码相比你用多个if else代码逻辑会更为清晰。

# array

```go
package main

import (
	"fmt"
)

func main() {
	var a [5]int
	a[4] = 100
	fmt.Println(a[4], len(a))

	b := [5]int{1, 2, 3, 4, 5}
	fmt.Println(b)

	var twoD [2][3]int
	for i := 0; i < 2; i++ {
		for j := 0; j < 3; j++ {
			twoD[i][j] = i + j
		}
	}
	fmt.Println("2d: ", twoD)
}
```

```
100 5
[1 2 3 4 5]
2d:  [[0 1 2] [1 2 3]]
```

## Key Points

1. 数组就是一个具有编号且长度固定的元素序列。
2. 对于一个数组，可以很方便地取特定索引的值或者往特定索引取存储值，然后也能够直接去打印一个数组。不过，在真实业务代码里面，很少直接使用数组，因为长度是固定的，用的更多的是切片。

# Slice

在Go语言中，"slice"是对数组的一种抽象表示。你可以把它想象成从数组中"切下"来的一片，所以被称为"slice"（切片）。切片是动态的，它们可以根据需要自动增长和缩小，这与数组的固定大小形成对比。切片提供了一个更灵活，更强大的接口来处理序列类型的数据。

```go
package main

import (
	"fmt"
)

func main() {
	s := make([]string, 3)
	s[0] = "a"
	s[1] = "b"
	s[2] = "c"
	fmt.Println("get:", s[2])   //c
	fmt.Println("len:", len(s)) //3

	s = append(s, "d")
	s = append(s, "e", "f")
	fmt.Println(s) // [a b c d e f]
	c := make([]string, len(s))
	copy(c, s)
	fmt.Println(c)

	fmt.Println(s[2:5]) // [c d e]
	fmt.Println(s[:5])  // [a b c d e]
	fmt.Println(s[2:])  // [c d e f]

	good := []string{"g", "o", "o", "d"}
	fmt.Println(good) // [g o o d]
}
```

## Key Points

1. 可以用make来创建一个切片，可以像数组一样去取值
2. 使用append来追加元素。注意append的用法，必须把append的结果返回给原数组。因为slice的原理实际上是它有一个它存储了一个长度和一个容量，加一个指向一个数组的指针，在执行append操作的时候，如果容量不够的话，会扩容并且返回新的 slice。
3. slice初始化的时候也可以指定长度。
4. slice拥有像python一样的切片操作，比如取出第二个到第五个位置的元素，不包括第五个元素。不过不同于python的是不支持负数索引。

# map

1. 可以用make创建一个空map

   ```go
   m := make(map[string]int)
   ```

   
