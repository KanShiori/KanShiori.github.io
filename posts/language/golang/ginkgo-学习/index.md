# Ginkgo 学习



## 1 概述
ginkgo 是一个 BDD 框架，Kubernetes 的 E2E 测试使用该框架实现集群的测试。

ginkgo 是集成在 Go 测试框架的，在目录下执行 `ginkgo bootstrap` 就会构建测试的入口：
```go
package ginkgo_test

import (
	"testing"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

func TestGinkgo(t *testing.T) {
    // Gomega 连接 Ginkgo
    // 使得 Gomega 断言能够通知到 Ginkgo
	RegisterFailHandler(Fail)

    // 测试入口
	RunSpecs(t, "Ginkgo Suite")
}
```

接着，我们在同目录下可以创建测试 Spec，然后通过 `ginkgo` 和 `go test` 命令就可以执行测试。


## 2 构建 Spec

#### 2.1 Describe Context It
为了更好的构建测试的结构，ginkgo 提供了三个接口来构建测试项的结构：
* Describe: 定义一个测试项
* Context: 定义一个测试项下的一种情况
* It: 真正执行的测试代码

以下面为例，Describe 定义了顶层的测试项与其中一个方面的测试项，两个 Context 为两个测试的情况，而 It 包含了具体的测试代码。
```go
// Root Describe
var _ = Describe("Book", func() {
	// Sub Describe
	Describe("Categorizing book length", func() {

		// Context1
		Context("With more than 300 pages", func() {
			It("should be a novel", func() {
				// 测试代码 + 断言
			})
		})

		// Context2
		Context("With fewer than 300 pages", func() {
			It("should be a short story", func() {
				// 测试代码 + 断言
			})
		})

	})

})
```

{{< admonition note Note>}}
也可以简单的将 Describe 与 Context 理解为将测试分类，复用一些描述情况。
{{< /admonition >}}

### 2.2 BeforeEach AfterEach
BeforeEach 会在每个 It 执行之前执行，一般用于测试进行数据的初始化。
```go
// Root Describe
var _ = Describe("Book", func() {
    var (
        // 通过闭包在 BeforeEach 和 It 之间共享数据
        longBook  Book
        shortBook Book
    )

    BeforeEach(func() {
        longBook = Book{
            Title:  "Les Miserables",
            Author: "Victor Hugo",
            Pages:  1488,
        }
 
        shortBook = Book{
            Title:  "Fox In Socks",
            Author: "Dr. Seuss",
            Pages:  24,
        }
    })

	// Sub Describe
	Describe("Categorizing book length", func() {

		// Context1
		Context("With more than 300 pages", func() {
			It("should be a novel", func() {
				// 测试代码 + 断言
			})
		})

		// Context2
		Context("With fewer than 300 pages", func() {
			It("should be a short story", func() {
				// 测试代码 + 断言
			})
		})

	})
})
```

相反，AfterEach 会在每个 It 执行完成后执行，用于数据的销毁。

### 2.3 JustBeforeEach JustAfterEach

JustBeforeEach 会在每个 It 执行之前执行，在对应的 BeforeEach 之后执行。

JustBeforeEach 出现主要是为了抽象出初始化数据的逻辑，使得只需要 BeforeEach 提供初始化数据来源。

例如下面例子，过 JustBeforeEach 就可以将原来的两次 BeforeEach 中的 NewBookFromJSON 调用，减少为一次。
```go
var _ = Describe("Book", func() {
    var (
        book Book
        err error
        json string
    )

    BeforeEach(func() {
        // 准备数据
        json = `{
            "title":"Les Miserables",
            "author":"Victor Hugo",
            "pages":1488
        }`
    })
 
    JustBeforeEach(func() {
        // 执行数据构建
        book, err = NewBookFromJSON(json)
    })
 
    Describe("loading from JSON", func() {
        Context("when the JSON parses succesfully", func() {
        })
 
        Context("when the JSON fails to parse", func() {
            BeforeEach(func() {
                // 覆盖数据
                json = `{
                    "title":"Les Miserables",
                    "author":"Victor Hugo",
                    "pages":1488oops
                }`
            })
        })
    })
})
```

对应的，JustAfterEach 在每个 It 之后调用，在 AfterEach 之前调用。

### 2.4 BeforeSuite AfterSuite

BeforeSuite 会在所有的测试执行前执行，AfterSuite 在所有的测试执行后执行：
```go
func TestBooks(t *testing.T) {
    RegisterFailHandler(Fail)
 
    RunSpecs(t, "Books Suite")
}
 
var _ = BeforeSuite(func() {
    dbClient = db.NewClient()
    err = dbClient.Connect(dbRunner.Address())
    Expect(err).NotTo(HaveOccurred())
})
 
var _ = AfterSuite(func() {
    dbClient.Cleanup()
})
```

### 2.5 SynchronizedBeforeSuite SynchronizedAfterSuite
SynchronizedBeforeSuite 用于开启并行运行后，指定在主进程执行的函数，以及在各个子进程执行的函数。函数都在运行测试之前执行。

例如，下面示例在子进程创建前，运行创建数据库。在各个子进程运行测试前，执行创建 client。
```go
var _ = SynchronizedBeforeSuite(func() []byte {
    // 在主进程中执行
    port := 4000 + config.GinkgoConfig.ParallelNode
 
    dbRunner = db.NewRunner()
    err := dbRunner.Start(port)
    Expect(err).NotTo(HaveOccurred())
 
    return []byte(dbRunner.Address())
}, func(data []byte) {
    // 在每个子进程中执行
    dbAddress := string(data)
 
    dbClient = db.NewClient()
    err = dbClient.Connect(dbAddress)
    Expect(err).NotTo(HaveOccurred())
})
```

相反的，通过 SynchronizedAfterSuite 来进程回滚。
```go
var _ = SynchronizedAfterSuite(func() {
    // 在每个子进程中执行
    dbClient.Cleanup()
}, func() {
    // 在主进程中执行
    dbRunner.Stop()
}) 
```

### 2.6 By
By 函数用于添加一些块文档，如果测试失败时会打印出来。可以将其理解为错误日志。
```go
var _ = Describe("Browsing the library", func() {
    BeforeEach(func() {
        By("Fetching a token and logging in")
    })
 
    It("should be a pleasant experience", func() {
        By("Entering an aisle")
    })
})
```


## 3 Spec Runner
### 3.1 Pending Spec
定义一个 Spec 或容器时，调用 P/X 前缀的接口，会定义一个 Pending Spec。默认不会执行
```go
PDescribe("some behavior", func() { ... })
PContext("some scenario", func() { ... })
PIt("some assertion")
PMeasure("some measurement")
 
XDescribe("some behavior", func() { ... })
XContext("some scenario", func() { ... })
XIt("some assertion")
XMeasure("some measurement")
```

通过命令行参数 *--noisyPendings=false* 可以将其执行。

### 3.2 Skiping Spec
通过 Skip 接口，你可以在代码中运行时跳过某个 Spec。
```go
It("should do something, if it can", func() {
    if !someCondition {
        // 跳过此 Spec，不需要 Return 语句
        Skip("special condition wasn't met")
    }
})
```

或者，你可以通过 *--skip=<regex>* 跳过运行匹配的 Spec。

### 3.3 Focused Specs
通过 F 前缀的接口，可以默认仅仅执行 Focused Spec
```go
FDescribe("some behavior", func() { ... })
FContext("some scenario", func() { ... })
FIt("some assertion", func() { ... })
```

或者，你可以在执行命令时通过 *--focus=<regexp>* 指定运行 Spec。

### 3.3 Parallel Specs
默认下，Ginkgo 执行测试是串行的，你可以通过 `ginkgo -p` 开启并行测试，Ginkgo 会自动创建适当数量的进程来并行执行。通过 `ginkgo -nodes=N` 也可以执行进程数量。

多个 go test 子进程会消费队列中的 Spec 并行执行。


## 4 Gomega
Gomega 库提供了断言的功能，

### 4.1 断言
Ω 与 Expect 都提供断言的功能，完全相同。

通过 Should/To 来进行 “应该” 逻辑的断言，通过 ShouldNot/NotTO/ToNot 进行 “不应该” 逻辑的断言。
```go
Expect(ACTUAL).Should(Equal(EXPECTED))
Expect(ACTUAL).To(Equal(EXPECTED))

Expect(ACTUAL).ShouldNot(Equal(EXPECTED))
Expect(ACTUAL).ToNot(Equal(EXPECTED))
Expect(ACTUAL).NoTo(Equal(EXPECTED))
```

Should 等函数后跟着 Matcher interface 变量，代表一个表达式。

### 4.2 Matcher

判断类型与值相等：
* Equal 使用 reflect.DeepEqual 进行比较。
* BeEquivalentTo 会先将 ACTUAL 转换为 EXPECTED 的类型，然后使用 reflect.DeepEqual 进行比较。
```go
Expect(ACTUAL).Should(Equal(EXPECTED))
Expect(ACTUAL).Should(BeEquivalentTo(EXPECTED))
```

接口相容：
* BeAssignableToTypeOf 仅仅用于判断能否将 EXPECTED 赋值给 ACTUAL。常用于判断 interface 是否满足。

空值与零值：
* BeNil 判断是否为 nil
* BeZero 判断是否为零值
```go
Expect(ACTUAL).Should(BeNil())
Expect(ACTUAL).Should(BeZero())
```

布尔值：
* BeTrue 判断为 true
* BeFalse 判断为 false
```go
Expect(ACTUAL).Should(BeTrue())
Expect(ACTUAL).Should(BeFalse())
```

error 处理：
* HaveOccurred 判断 error 为 nil
* Succeed 判断 error 不为 nil
* MatchError 以判断 error 或者 string 是否相同
```go
Expect(err).ShouldNot(HaveOccurred())
Expect(err).Should(Succeed())
Expect(err).Should(MatchError("an error")) //asserts that err.Error() == "an error"
Expect(err).Should(MatchError(SomeError)) //asserts that err == SomeError (via reflect.DeepEqual)
```

channel 处理：
* BeClosed 判断 channel 已经关闭，会读取 channel 数据来判断
* Receive 判断 channel 中能否读取到数据
* BeSent 判断 channel 能否无阻塞发送消息
```go
Expect(ch).Should(BeClosed())
Expect(ch).Should(Receive(<optionalPointer>))
Expect(ch).Should(BeSent(VALUE))
```

文件处理：
* BeAnExistingFile 判断文件/目录是否存在
* BeARegularFile 判断是否为普通文件
* BeADirectory 判断是否为目录
```go
Expect(ACTUAL).Should(BeAnExistingFile())
Expect(ACTUAL).Should(BeARegularFile())
Expect(ACTUAL).Should(BeADirectory())
```

字符串处理：
* ContainSubstring 判断是否包含子串
* HavePrefix 判断是否包含前缀
* HaveSuffix 判断是否包含后缀
* MatchRegexp 进行正则匹配
```go
Expect(ACTUAL).Should(ContainSubstring(STRING, ARGS...))
Expect(ACTUAL).Should(HavePrefix(STRING, ARGS...))
Expect(ACTUAL).Should(HaveSuffix(STRING, ARGS...))
Expect(ACTUAL).Should(MatchRegexp(STRING, ARGS...))
```

JSON/XML/YAML 处理：
* MatchJSON 判断 JSON 是否相同
* MatchXML 判断 XML 是否相同
* MatchYAML 判断 YAML 是否相同

集合（string、array、map、chan、slice）处理：
* BeEmpty 判断为空
* HaveLen 判断长度
* HaveCap 判断容量
* ContainElement 判断是否包含元素
* BeElementOf 判断值是否在集合其中一个
* ConsistOf 判断两个集合元素是否相同，不考虑顺序
* HaveKey 判断 map 是否包含指定的 key
* HaveKeyWithValue 判断 map 是否包含指定的 key/value
```go
ints := []int{1, 2, 3}
m := map[string]string{}
Expect(ints).Should(BeEmpty())
Expect(ints).Should(HaveLen(1))
Expect(ints).Should(HaveCap(2))
Expect(ints).Should(ContainElement(3))
Expect(1).Should(BeElementOf(ints)
Expect(ints).Should(ConsistOf(1, 2, 3))
Expect(ints).Should(ConsistOf([]int{3, 2, 1}))
Expect(m).Should(HaveKey("a"))
Expect(m).Should(HaveKeyWithValue("a", "b"))
```

数字/时间处理：
* BeNumerically 进行数值的比较，包含以下运算符:
  * == 判断数值是否相等
  * ~ 判断数值是否相似（一定范围内）
  * > >= < <= 比较大小
* BeBetween 判断是否在范围内
* BeTemporally 进行 time.Time 类型比较，方式与 BeNumerically 相同
```go
a := 1
b := 2
Expect(a).Should(BeNumerically("==", b))
Expect(a).Should(BeNumerically("~", b, 1))
Expect(a).Should(BeNumerically(">", b))
Expect(a).Should(BeNumerically(">=", b))
Expect(a).Should(BeNumerically("<", b))
Expect(a).Should(BeNumerically("<=", b))
Expect(a).Should(BeBetween(0, 10))
```

panic：
* Panic 判断函数是否会发生 panic
```go
Expect(func(){}).Should(Panic())
```

逻辑组合：
* SatisfyAll/And 进行逻辑与的组合
* SatisfyAny/Or 进行逻辑或的组合
```go
Expect(msg).To(And(Equal("Success"), MatchRegexp(`^Error .+$`)))
Expect(msg).To(Or(Equal("Success"), MatchRegexp(`^Error .+$`)))
```
