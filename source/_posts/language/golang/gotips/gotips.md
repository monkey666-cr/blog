---
title: "gotips 0-20"
date: {{ date }}
categories:
    - 语言
    - Golang
    - gotips
---

## 摘要

每日读书笔记

## 一行代码测量执行时间

```golang
package main

import (
	"fmt"
	"time"
)

func main() {
	defer TrackTime(time.Now())

	time.Sleep(500 * time.Millisecond)
}

func TrackTime(pre time.Time) time.Duration {
	elapsed := time.Since(pre)
	fmt.Println("elapsed:", elapsed)

	return elapsed
}
```

## 多阶段defer

```golang
package main

import "fmt"

func main() {
	defer MultistageDefer()()

	fmt.Println("Main function called")
}

func MultistageDefer() func() {
	fmt.Println("Run initialization")

	return func() {
		fmt.Println("Run cleanup")
	}
}
```

## 预分配切片以提高性能

```golang
package main

func main() {
	// MISTAKE
	a := make([]int, 5)
	a = append(a, 1) // [0, 0, 0, 0, 0, 1]

	// BETTER
	b := make([]int, 0, 5)
	b = append(b, 1) // [1]
}
```

## 切片转换为数组

```golang
package main

import "fmt"

func main() {
	slice := []int{1, 2, 3, 4, 5}
	var array [5]int

	copy(array[:], slice)

	// go 1.17
	c := []int{0, 1, 2, 3, 4, 5}
	d := *(*[3]int)(c[0:3])
	fmt.Println(d)

	// go version >= 1.20
	a := []int{0, 1, 2, 3, 4, 5}
	b := [3]int(a[0:3])
	fmt.Println(b)
}
```

## 方法链

```golang
package main

import "fmt"

type Person struct {
	Name string
	Age  int
}

func (p *Person) AddAge() *Person {
	p.Age++
	return p
}

func (p *Person) Rename(name string) *Person {
	p.Name = name
	return p
}

func main() {
	// NOT BAD
	p := Person{Name: "Alex", Age: 30}
	p.AddAge()
	p.Rename("hello")
	fmt.Println(p)

	// BETTER
	a := (&Person{Name: "Alex", Age: 30}).
		AddAge().
		Rename("Hello")

	fmt.Println(a)
}
```

## 下划线导入

下划线导入时，会执行包的init方法。

## 包裹错误

通常使用`fmt.Errorf和%w把一个错误包裹到另一个错误里`

```golang
package main

import (
	"errors"
	"fmt"
)

func error_example() error {
	err := errors.New("error from Func1")

	return fmt.Errorf("error from Func2 %w", err)
}

func main() {
	err := error_example()

	fmt.Println(err)
}
```

Go1.20版本提供了`errors.Join()`：

```golang
package main

import (
	"errors"
	"fmt"
)

func Func1() error {
	err := errors.New("error from Func1")

	return err
}

func Func2() error {
	err := Func1()
	if err != nil {
		return errors.Join(err, errors.New("error from Func2"))
	}

	return nil
}

func main() {
	err := Func2()

	fmt.Println(err)
}
```

## 编译时接口检查

```golang
package main

type Buffer interface {
	Write(p []byte) (n int, err error)
}

type StringBuffer struct{}

// 故意写错
func (s *StringBuffer) Writee(p []byte) (n int, err error) {
	return 0, nil
}

// 编译器报错
var _ Buffer = (*StringBuffer)(nil)

func main() {

}
```

## 避免裸露参数

```golang
package main

func printInfo(name string, isLocal bool, done bool) {

}

func main() {
	// NOT BAD
	printInfo("foo", true, true)

	// BETTER
	printInfo("foo",
		true, /* isLocal */
		true, /* done */
	)
}
```

## 数字分隔符

```golang
// NOT BAD
const oneBillion = 10000000000

// BETTER
const onBillion = 1_000_000_000

// float
const pi = 3.141_592_653
```

## 使用`crypto/rand`生成密钥，避免使用`math/rand`

`math/rand`是伪随机

```golang
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func Key() string {
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	buf := make([]byte, 16)

	for i := range buf {
		buf[i] = byte(r.Intn(256))
	}

	return fmt.Sprintf("%x", buf)
}

func main() {
	k := Key()
	fmt.Println(k)
}
```

```golang
package main

import (
	"crypto/rand"
	"fmt"
)

func Key() string {
	buf := make([]byte, 16)

	_, err := rand.Read(buf)
	if err != nil {
		panic(err)
	}

	return fmt.Sprintf("%x", buf)
}

func main() {
	k := Key()
	fmt.Println(k)
}
```

## 使用空切片还是更好的nil切片

`var t []int`这个方法声明了一个int类型的切片t，并没有对其进行初始化。这个时候这个切片被认为是`nil`。这意味着它没有指向任何的底层数组。长度和容量都是0。

`t := []int{}`和`var`声明的切片不一样，这个切片不是nil的。这个切片的指向了一个底层数组，但这个数组没有任务元素。

使用JSON的时候，nil切片的值对应的是none，空切片是空数组。

对nil切片进行`len`, `append`, `range`并不会报错


```golang
package main

import "fmt"

func main() {
	var s []int
	fmt.Println(len(s))

	s = append(s, 1)
	s = append(s, 2)
	s = append(s, 3)
	fmt.Println(s)

	for _, item := range s {
		fmt.Println(item)
	}

	fmt.Println(len(s))
}
```

## 错误信息不要大写或者以标点结尾

因为错误信息经常被包裹或者合并到其他错误信息里。如果一条错误信息以大写字母开头，那么当它出现在句子中间的时候，看起来很奇怪。

## 什么时候使用空白导入和点导入

空包导入是为了执行init方法。点导入时可以直接调用方法。

## 不要返回-1或者nil来表示错误

解决方案是：多返回值

函数可以返回其通常的结果以及额外的值来表示操作是否成功。使代码更加清晰。

## 尽快返回，避免多层嵌套

提前处理错误，当出现一个错误时，立刻处理它。使用return、break、continue等语句停止当前操作的执行。

## 在使用者的包中定义接口，而不是提供方的包中定义接口

## 传递值，而不是指针

- 固定大小的类型
- 不变性和清晰度
- 小型或不太可能增长的类型
- 传递值的速度很快，而且很少比传递指针慢
- 将传递值设为默认值

## 定义方法时，优先使用指针作为接收器（receiver）

## 使用结构体或变长参数简化函数签名

- 结构体作为参数
- 变长参数

```golang
package main

type ServiceConfig struct {
	ssl bool
}

type ServiceOption func(*ServiceConfig)

func WithSSL(ssl bool) ServiceOption {
	return func(sc *ServiceConfig) {
		sc.ssl = ssl
	}
}

func ConnectToDatabase(options ...ServiceOption) {
	cfg := ServiceConfig{}
	for _, option := range options {
		option(&cfg)
	}
}

func main() {
	ConnectToDatabase(WithSSL(true))
}
```

## 省略getter方法的Get前缀

```golang
package main

type Book struct {
	title string
}

func (b *Book) Title() string {
	if len(b.title) == 0 {
		return "Unknow"
	}

	return b.title
}

func (b *Book) SetTitle(newTitle string) {
	b.title = newTitle
}
```

## 避免命名中的重复

- 包名与导出符号名称
- 变量名与类型，不要在变量名中重复类型名称
- 避免重复归结于上下文

## goroutines 之间信号传递时，使用`chan struct{}`而不是`chan bool`

因为`struct{}`不占内存。

## 使用空标识符(_)明确忽略值，而不是无声的忽略它们

## 原地过滤

```golang
package main

import "fmt"

func isOdd(num int) bool {
	return num%2 == 0
}

func main() {
	// 需要多申请一块内存保存过滤之后的数据
	var filtered []int
	var numbers = []int{1, 2, 3}

	for _, num := range numbers {
		if isOdd(num) {
			filtered = append(filtered, num)
		}
	}

	fmt.Println(filtered)
}
```

可以将numbers的切片赋值给filtered，直接复用底层数据

```golang
package main

import "fmt"

func isOdd(num int) bool {
	return num%2 == 0
}

func main() {
	var numbers = []int{1, 2, 3}
	filtered := numbers[:0]

	for _, num := range numbers {
		if isOdd(num) {
			filtered = append(filtered, num)
		}
	}

	fmt.Println(filtered)
}
```

## 将多个if-else语句转换为switch

```golang
package main

import (
	"fmt"
	"time"
)

func main() {
	t := time.Now()
	switch {
	case t.Hour() < 12:
		fmt.Println("It's before noon")
	default:
		fmt.Println("It's after noon")
	}
}
```

## 避免使用context.Background，使协程具备承诺性

这里的承诺性是指协程运行态状态应该是确定的，而不是一直无限期地一直运行下去。

一般来说，有两种方法可以使协程具有承诺性：取消和超时

```golang
package main

import (
	"context"
	"errors"
	"time"
)

func main() {
	ctx := context.Background()
	ctx, cancelFunc = context.WithTimeout(ctx, time.Duration(10))
	ctx, cancelFunc = context.WithTimeoutCause(ctx, time.Duration(10), errors.New("custom message"))

	ctx, cancelCauseFunc = context.WithCancel(ctx)
	ctx, cancelCauseFunc = context.WithCancelCause(ctx)

	ctx, cancelFunc = context.WithDeadline(ctx, time.Now())
	ctx, cancelFunc = context.WithDeadlineCause(ctx, time.Now(), errors.New("custom error"))
}
```

## 使用context.WithoutCancel()继续上下文操作

```golang
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	parentCtx, cancelFunc := context.WithCancel(context.Background())
	childCtx, _ := context.WithCancel(parentCtx)

	go func(ctx context.Context) {
		<-ctx.Done()
		fmt.Println("Child context canceled")
	}(childCtx)

	cancelFunc()

	time.Sleep(time.Second)
}
```

有一些场景当超时之后，需要继续执行后续的一写操作，比如记录当前操作的日志。

```golang
package main

import (
	"context"
	"fmt"
	"time"
)

func asyncTask(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println("异步任务收到取消信号：", ctx.Err())
			return
		default:
			fmt.Printf("异步任务执行中, traceID: %v\n", ctx.Value("traceID"))
			time.Sleep(1 * time.Second)
		}
	}
}

func main() {
	parentCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	parentCtx = context.WithValue(parentCtx, "traceID", "12345")
	defer cancel()

	childCtx := context.WithoutCancel(parentCtx)

	go asyncTask(childCtx)

	<-parentCtx.Done()
	fmt.Println("父context已取消，但异步任务继续运行...")
	time.Sleep(3 * time.Second)
}
```

## Scheduling functions after context cancellation with context.AfterFunc

```golang
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	stop := context.AfterFunc(ctx, func() {
		fmt.Println("Cleanup operations after context is done")
	})

	time.Sleep(3 * time.Second)

	if stopped := stop(); stopped {
		fmt.Println("Callback was stopped before execution")
	}
}
```

## 尽量不要使用panic()

这段代码会panic

```golang
package main

import (
	"fmt"
	"time"
)

func panicFunc() {
	panic("I'm panicking!")
}

func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered from panic")
		}
	}()

	go panicFunc()

	time.Sleep(time.Second)
}
```

## 以context开头，已options结尾，并且总是用error来关闭

## 转换字符串优先使用strconv而非fmt

```golang
package main

import (
	"fmt"
	"strconv"
)

func main() {
	a := 10

	fmt.Println(strconv.Itoa(a))
}
```

## 全局变量前加下划线前缀，声明在顶层的变量和常量可以在在它们所说的整个包中被访问。

优先避免无意修改变量的值

## 使用未导出的空结构体作为上下文键

```golang
type traceIDKey struct{} // 空结构体不占内存

func WithTraceID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, traceIDKey{}, id)
}

func GetTraceID(ctx context.Context) string {
    if v := ctx.Value(traceIDKey{}); v != nil {
        return v.(string)
    }
    return ""
}
```

## 使用`fmt.Errorf`使错误信息清晰明了，不要让他们过于赤裸

使用`fmt.Errorf`和`%w`

```golang
func doSomething() error {
	err := someOperation()
	if err != nil {
		return fmt.Errorf("failed to do someOperation: %w", err)
	}
	return nil
}
```

## 避免在循环中使用defer，否则可能会导致内存溢出

在Go中使用defer时，我们一般希望defer后面的函数能在当前函数返回之前被执行。

## 在使用defer时处理错误以防止静默失败

```golang
func OpenFile(path string) (err error) {
	file, err := os.Open(path)
	if err != nil {
		return err
	}

	defer func() {
		if cerr := file.Close(); cerr != nil {
			err = errors.Join(err, cerr)
		}
	}()
}
```

```golang
defer closeWithError(&err, file)

func closeWithError(err *error, closer io.Closer) {
	*err = errors.Join(*err, closer.Close())
}
```

## 将结构体中的字段按大到小的顺序排列

结构体中字段顺序会影响到结构体自身的大小，可以利用这一点来优化内存使用

```golang
package main

type StructA struct {
	A byte  // 1-byte
	B int32 // 4-byte
	C byte  // 1-byte
	D int64 // 8-byte
	E byte  // 1-byte
}

type OptimizedStructA struct {
	D int64 // 8-byte
	B int32 // 4-byte
	A byte  // 1-byte
	C byte  // 1-byte
	E byte  // 1-byte
}
```

结构体StructA使用了32字节，而OptimizedStructA进需16字节

## 单点错误处理，降低噪音

## 优雅关闭程序

- 不接受新请求：服务停止接受新的请求。
- 完成正在进行打分任务。等待当前处理的任务达到逻辑上的停止点。
- 资源清理：释放诸如数据库连接、打开文件、网络连接等资源。
  
```golang
package main

import (
	"context"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"golang.org/x/sync/errgroup"
)

func main() {
	ctx, cancel := signal.NotifyContext(
		context.Background(), os.Interrupt, syscall.SIGTERM,
	)
	defer cancel()

	server := http.Server{Addr: ":8080"}

	g, gCtx := errgroup.WithContext(ctx)
	g.Go(func() error { return server.ListenAndServe() })
	g.Go(func() error {
		<-gCtx.Done()
		time.Sleep(3 * time.Second)
		return server.Shutdown(context.Background())
	})

	if err := g.Wait(); err != nil {
		fmt.Printf("exit with: %v\n", err)
	}
}
```

## 有意地使用Must函数来停止程序

这类函数有一个特定的命名模式，使用`Must`开头，这就是提醒调用者，如果程序没有按照预期执行的话就会导致panic

通常情况下不应失败的初始化任务，例如：在应用程序开始设置包级变脸、设置正则表达式、连接数据库等。

Must函数实在初始化和编写单元测试时候的工具，用于处理难以预料的情况

```golang
func mustDecodeJSON(t *testing.T, jsonStr string, v any) {
	t.Helper()
	if err := json.Unmarshal([]byte(jsonStr), v); err != nil {
		t.Fatalf("mustDecodeJSON failed: %v", err)
	}
}
```

## 始终管理协程的声明周期

```golang
func Job(ctx context.Context) {
	for {
		select {
		case <- ctx.Done():
			return
		default:
			time.Sleep(10 * time.Second)
		}
	}
}
```

另一个可能导致Golang协程永远卡主的场景：

```golang
func worker(jobs <-chan int) {
	for job := range jobs {
		// ...
	}
}

jobs := make(chan int)
go worker(jobs)
```

我们可能任务一旦作业通道关闭，就很容易确定协程何时结束。

但是工作通道何时会关闭呢？

可能会犯一个错误，在没有关闭通道而是直接从函数中返回，这会导致协程无限期的挂起，进而引发内存泄漏。

务必确保协程的启动和停止时机是显而易见的，并务必将上下文传递给长时间运行的任务。

## 避免在switch语句的case中使用break，除非与标签一起使用

Go的switch语句中的每个case自带一个隐式的break。

```golang
loop:
	for {
		switch x {
		case "A":
			break loop
		}
	}
```