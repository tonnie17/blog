+++
title = "go单元测试规范"
date = 2019-07-27T00:00:00+08:00
tags = ["go"]
categories = [""]
draft = false
+++

## 测试原则

1. **编写可测试的代码。**

编写可测试的代码意味着在编写代码时就要考虑到这段代码是否易于测试。

例如对于以下这段代码：

```go
func NewHouse() *House {
    kitchen := new(Kitchen)
    bedroom := new(Bedroom)
    return &House{
        kitchen: kitchen,
        bedroom: bedroom,
    }
}
```

上面这段代码中在构造函数中创建对象，这样做导致了无法用多态的手段来替换House中Kitchen或Bedroom的行为，如果Kitchen中包含了某些昂贵的操作，比如数据库访问，那么这段代码将变得不容易测试。反之，使用依赖注入的方式将使得代码变得更加易于测试：

```go
func NewHouse(Kitchen *k, Bedroom *b) *House {
    return &House{
        kitchen: k,
        Bedroom: b,
    }
}
```

此外，还可以借助interface{}来实现编写可测试的代码，例如对于下面这段代码：

```go
func populateInfo(fetcher HttpResponseFetcher, parsedInfo *Info) error {
    response, err := fetcher.Fetch("http://example.com/info")

    if err == nil {
        err = json.Unmarshal(response, parsedInfo)

        if err == nil {
            return nil
        }
    }

    return err
}
```

它包含了一个对外的网络请求访问，但如果我们的测试更加关注于程序的逻辑，那么这个网络的操作就变得与测试无关，这时候我们便可以通过接口的方式来替代传入对象：

```go
type HttpResponseFetcher interface {
    Fetch(url string) ([]byte, error)
}
```

只要对象实现了`HttpResponseFetcher`这个接口，我们便可以更自由地控制程序行为：

```go
type stubFetcher struct{}

func (fetcher stubFetcher) Fetch(url string) ([]byte, error) {
    if strings.Contains(url, "/info") {
        return infoOutput, nil
    }

    if strings.Contains(url, "/status") {
        return statusOutput, nil
    }

    return nil, errors.New("Don't recognize URL: " + url)
}
```

参考Google的[Guide-Writing Testable Code](https://link.zhihu.com/?target=http%3A//misko.hevery.com/attachments/Guide-Writing%20Testable%20Code.pdf)

**2. 在代码上主干分支之前必须包含测试代码。**

**3. 提交代码前确保测试全部通过。**

**4. 良好的测试可描述代码如何组织。**

好的代码可以通过测试来直接描述程序行为，如果无法通过测试来看出其功能，考虑进行更细的拆分。

**5. 单元测试应全自动化，避免人工介入。**

单元测试中避免使用`fmt.Print`等方式来人肉验证，应使用`*testing.T`对象或`assert`进行验证。

**6. 每个对外暴露的接口和主流程都应拥有测试用例。**

除非函数特别复杂，否则大部分情况只需要关心程序对外暴露接口的测试。

**7. 对不可测的代码重构，使之变得可测。**

参考[重构改善既有代码的设计](https://link.zhihu.com/?target=https%3A//www.kancloud.cn/sstd521/refactor)。

**8. 每个测试用例都是一座孤岛，不依赖其他测试用例。**

测试用例的执行结果不应该依赖于用例的执行顺序，也不应依赖状态化的全局变量，相反它们是可独立执行的，只依赖程序功能。

**9. 确保测试不受环境影响，对于无法使用的外部依赖(网络，DB)可以使用Mocking来做依赖替换。**

- [go-sqlmock](https://link.zhihu.com/?target=https%3A//github.com/DATA-DOG/go-sqlmock)
- [miniredis](https://link.zhihu.com/?target=https%3A//github.com/alicebob/miniredis)
- [golang/mock](https://link.zhihu.com/?target=https%3A//github.com/golang/mock)

**10. 测试应完全通过Race Detection。**

通过Race Detector的检测，确保程序在运行`go test -race`的情况下也能通过测试。

**11. 测试前后保持数据环境清理干净。**

在测试用例中创建的一些临时文件或记录，尽量在用例结束前进行清理，如：

```go
func TestLogging(t *testing.T) {
    tempDir := makeTempDir(tempPath)
    defer os.RemoveAll(tempDir)
    testLogging(tempDir)
}
```

**12. 保证足够高的覆盖率。**

## 书写规范

- 一个测试文件对应另一个被测试文件，放在同一目录(包)下，命名规则为被测试文件后加上`_test`后缀，如：`split.go`对应`split_test.go`。
- 单元测试函数名称以`Test`作为前缀，参数为`*testing.T`，如：`TestSplit(t *testing.T)`。
- 示例测试函数名称以`Example`作为前缀，如：`ExampleSplit()`。
- 性能测试函数名称以`Benchmark`作为前缀，参数为`*testing.B`，如：`Benchmark(b *testing.B)`。
- 对于单元测试，至少包括一个校验，可使用[testify](https://link.zhihu.com/?target=https%3A//github.com/stretchr/testify)进行断言。
- 对于Test fixtures，建立一个`testdata`目录来存放数据。

## 表驱动测试

- 测试包含自己的子测试用例列表。
- 用`name`字段来区分每个子测试用例的含义。
- 每个子用例对应一个使用场景。

示例：

split.go

```go
// Split slices s into all substrings separated by sep and
// returns a slice of the substrings between those separators.
func Split(s, sep string) []string {
    var result []string
    i := strings.Index(s, sep)
    for i > -1 {
        result = append(result, s[:i])
        s = s[i+len(sep):]
        i = strings.Index(s, sep)
    }
    return append(result, s)
}
```

split_test.go

```go
func TestSplit(t *testing.T) {
    tests := map[string]struct {
        input string
        sep   string
        want  []string
    }{
        "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
        "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got := Split(tc.input, tc.sep)
            if !reflect.DeepEqual(tc.want, got) {
                t.Fatalf("expected: %v, got: %v", tc.want, got)
            }
        })
    }
}
```

BDD:

[Goconvey](https://link.zhihu.com/?target=https%3A//github.com/smartystreets/goconvey)

[Ginko](https://link.zhihu.com/?target=https%3A//github.com/onsi/ginkgo)
