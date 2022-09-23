# Golang

生成Go项目目录

```go
go mod init 项目名
```

## 包

import "fmt"

import (

​		"fmt"

)

## 对象声明

1. var a int 默认值0
2. var a int  = 10
3. var a = 100 不指定类型，通过指自动匹配数据类型（不推荐）
4. a := 100 （常用）省略var和类型声明 并直接赋值，该方式不能声明全局变量

多变量声明

单行

```go
var aa, bb int = 100, 200
var cc, dd = 100, "dd"
```

多行

```go
var (
		a int = 100
		b bool = false
)
```

### 常量

const

```go
const a = 10
const a int = 10
```



自增常量 iota，只能配合const使用

```go
const i = iota
```

```go
const (
	USER = iota // iota自增, iota = 0
	MANAGER // iota = 1
	TEACHER // iota = 2
	STUDENT // iota = 3
)
```

## 方法返回

```go
// 无参无返回值
func f1() {
}

```

```go
// 有参有返回值
func f2(a int) int {
	println(a)
	b := 1
	return b
}
```

```go
// 多返回值
func f3(a int) (int, string) {
	fmt.Println(a)
	return 555, "aaa"
}
```

```go
// 有名称返回值
func f4(a int) (i1 int, i2 int) {
	fmt.Println(a)
	i1 = 10
	i2 = 20
	return
}
```

```go
// 有名称返回值2
func f5(a int) (i1, i2 int) {
	fmt.Println(a)
	i1 = 10
	i2 = 20
	return
}
```

## 构造体（对象）

type User struct {

​		Id int

​		Name string

}



## 值类型方法

func (s User) changeUser() {

​		s.id = 1

​		s.name = "Jone"

}



## 指针类型方法

func (s <font color="red">*</font>User) changeUser() {

​		s.id = 1

​		s.name = "Jone"

}