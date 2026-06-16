+++
date = '2026-05-30T20:16:35+08:00'
draft = false
title = '原来用Go写测试这么简单'
+++

在 Go 的开发过程中，可能都碰到过：

- 函数返回值中明确定义了 error 返回类型，但调用方直接用 `_` 忽略，虽然程序不报错，但这就等于在程序中埋了一个不知道什么时候会被引爆的雷
- 写了一个折扣计算函数，只跑了正常输入的情况，就编译提交发布。结果线上的折扣计算出现各种奇怪的结果，只能紧急上线修复。

这就像是一架飞机，已经有了一些问题，却没有进行维修检查，继续飞行，后果是可想而知的。

同样，为了保证我们的代码在上线后能够正确运行，在发布前也需要进行维修检查，在 Go 中，通过测试来完成这类检查。

提到写测试，可能有这样的反应：

- 写完业务代码还写测试，这不增加工作量吗
- 又要学习一种新的框架或库，哪有时间去学习

这些情况我也遇到过，但写过 Go 测试之后，我发现，在 Go 中写测试很简单：一个文件，一个函数，一条命令。

---

## 三步跑通 Go 测试

### 折扣计算函数

在测试之前，先来创建一个测试项目：

```bash

mkdir price-demo && cd price-demo

go mod init price-demo

```

写一个 `price.go` 文件，添加折扣计算函数：

```go

package price

import "errors"

func ApplyDiscount(originalPrice int, discountPercent int) (int, error) {

    if originalPrice < 0 {
        return 0, errors.New("price must not be negative")
    }

    if discountPercent < 0 || discountPercent > 100 {
        return 0, errors.New("discount must be between 0 and 100")
    }

    return originalPrice * (100 - discountPercent) / 100, nil
}

```

`ApplyDiscount` 函数接受两个参数原价和折扣百分比，输出折扣价和一个错误。处理三种场景：

- 原价为负数，返回错误
- 折扣超出范围，返回错误
- 计算折扣价并返回结果

### 创建测试文件写测试函数

写完折扣计算函数，再给它加个测试文件。在同目录下创建 一个名为 `price_test.go` 的文件：

```go
// price_test.go
package price

import "testing"

func TestApplyDiscount_NormalCase(t *testing.T) {

    got, err := ApplyDiscount(200, 20)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    want := 160
    if got != want {
        t.Errorf("ApplyDiscount(200, 20) = %d, want %d", got, want)
    }
}
```

测试文件名称 `price_test.go`，折扣函数文件为 `price.go`，测试文件多了一个 `_test`，可能有人会问，是不是 Go 语言对测试文件的命名就做了这样的规定？

其实，这种写法只是 Go 社区的一种推荐用法，实际在运行这些测试的时候，Go 编译器只编译每个包下的以`_test` 结尾的文件。

因此测试文件的名称必须以 `_test` 结尾，但社区使用方法更能明确的表达测试意图。

说完测试文件，再来看看测试函数：
- **函数名：** `TestApplyDiscount_NormalCase`：`Test` 开头是硬性要求，后面的 `_NormalCase` 是场景描述，方便识别。注意，如果函数以 `test` 开头，Go 会无视该测试用例，跳过不执行，结果中也不会包含任何该函数的信息。
- **检查 error**：Go 函数常返回 `(result, error)`，测试中先检查 err 是否为 nil。
- **`t.Fatalf`**：如果 err 不为 nil，后面的比较没有意义（`got` 的值不可靠），所以用 `Fatalf` 立即停止当前测试。
- **`t.Errorf`**：标记失败但继续执行。如果后面还有其他测试用例，会继续执行

### 运行

```bash
go test
```

输出：

```
PASS
ok price-demo/price 0.002s
```

运行测试，会看到对应的测试结果。输出了两行：
- 第一行 PASS 表示测试运行的结果
- 第二行是每个模块的测试状态，其中 `ok` 表示当前模块运行通过，`price-demo/price` 对应的项目下的具体的模块，当前例子就是 `price` 模块， `0.002s` 运行测试时间

如果测试运行失败，会看到什么？把上面测试用例中 `want := 160` 故意改成 `want := 150`，再跑一次：

```
--- FAIL: TestApplyDiscount_NormalCase (0.00s)
 price_test.go:11: ApplyDiscount(200, 20) = 160, want 150
FAIL
exit status 1
FAIL myproject/price 0.002s
```

到此，写完了一个完整的 Go 测试：**一个文件、一个函数、一条命令。**

---

## 独立函数模式:一个场景一个函数

在 `ApplyDiscount` 函数中，"正常打折"只是其中的一种情况，还有多种异常情况：
- 非法输入：负数价格、越界折扣
- 边界值：0% 折扣、100% 折扣

如何组织不同场景的测试？Go 语言给我们提供了一种模式：**独立函数模式**。

非法输入——验证函数能正确拒绝非法输入：

```go
func TestApplyDiscount_NegativePrice(t *testing.T) {
    _, err := ApplyDiscount(-100, 20)
    if err == nil {
        t.Error("expected error for negative price, got nil")
    }
}

func TestApplyDiscount_DiscountOver100(t *testing.T) {
    _, err := ApplyDiscount(200, 150)
    if err == nil {
        t.Error("expected error for discount > 100, got nil")
    }
}
```

边界值——验证极端值下的行为：

```go
func TestApplyDiscount_ZeroDiscount(t *testing.T) {
    got, err := ApplyDiscount(200, 0)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if got != 200 {
        t.Errorf("ApplyDiscount(200, 0) = %d, want 200", got)
    }
}

func TestApplyDiscount_FullDiscount(t *testing.T) {
    got, err := ApplyDiscount(200, 100)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if got != 0 {
        t.Errorf("ApplyDiscount(200, 100) = %d, want 0", got)
    }
}
```

运行 `go test -v`，看到完整的场景清单：

```
=== RUN TestApplyDiscount_NormalCase
--- PASS: TestApplyDiscount_NormalCase (0.00s)
=== RUN TestApplyDiscount_NegativePrice
--- PASS: TestApplyDiscount_NegativePrice (0.00s)
=== RUN TestApplyDiscount_DiscountOver100
--- PASS: TestApplyDiscount_DiscountOver100 (0.00s)
=== RUN TestApplyDiscount_ZeroDiscount
--- PASS: TestApplyDiscount_ZeroDiscount (0.00s)
=== RUN TestApplyDiscount_FullDiscount
--- PASS: TestApplyDiscount_FullDiscount (0.00s)
PASS
ok myproject/price 0.002s
```

> `go test -v` 会输出所有测试函数的运行结果：一份场景清单，只想看到测试是否通过，执行 `go test`

可以清看到：
- **每个函数独立运行，互不干扰**——一个测试失败不影响其他测试
- **函数名本身就是文档**——`TestApplyDiscount_NegativePrice` 看到函数名字就能明白测的是什么

这是 Go 测试最基础的组织方式——在 2016 年 `t.Run` 出现之前，这是唯一的写法。到今天，当各场景的测试逻辑差异较大时，它仍然是最清晰的选择。

但**独立函数模式**也有它的不足之处，试想一下这种情况：把测试对象从 `ApplyDiscount` 换成另外一个函数，恰巧这个函数比较复杂，在应用这种模式时，会发现编写的测试函数结构几乎一模一样——调用函数、检查 error、比较结果。会出现大量的重复代码，Go 社区也有更好的组织模式来解决这些问题，后面会继续介绍这些模式。

---

## 小结

在 Go 中写测试，其实只需要遵循简单的命名规范，利用标准库就能轻松完成，就这么简单，没有其他依赖。

通过独立函数模式，可以组织各种场景的不同测试，保证测试用例之间的绝对隔离，同时每个用例都有一个清晰的名字，函数名就是文档。

不过，我们也看到了，在测试场景的数量比较多的情况下，独立测试模式就会出现重复的结构。在实践中你们是怎么处理的呢？

