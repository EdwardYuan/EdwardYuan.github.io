---
layout: post
title:  "老项目怎么迁移到 Go 的 JSON v2"
date:   2016-11-11
categories: go
---

写 Go 1.25 的时候，看过一次 JSON v2。当时还要开 `GOEXPERIMENT=jsonv2`。到了 Go 1.27，`encoding/json/v2` 已经正式进入标准库。旧的 `encoding/json` 也换成了基于 v2 的实现，保留 v1 的行为。也就是说，升级 Go 版本和迁移到 v2，可以分开做。 

真要迁移，需要先看接口上实际收发的 JSON。有些行为在 Go 代码里没写明，下游却可能已经用了好几年。

## 先看输出，再看输入

比如一个普通的响应结构体：

```go
type Pet struct {
    Name      string   `json:"name"`
    Nicknames []string `json:"nicknames"`
}

p := Pet{Name: "Remi"}
```

v1 输出 
```{"name":"Remi","nicknames":null}```

v2 输出 
```{"name":"Remi","nicknames":[]}```

`nil` map 也有类似的变化，默认从 `null` 变成 `{}`。对这个例子来说，Go 里的 `Nicknames` 仍然是 `nil`，变的是客户端收到的值。如果客户端用 `null` 表示「还没设置」，用 `[]` 表示「设置过，列表为空」，直接切换就会丢掉这个区别。需要保留原来的输出，可以给这个字段加 `format:emitnull`，也可以在编码时传 `jsonv2.FormatNilSliceAsNull(true)`。 

输入侧更容易漏掉的是大小写。上面的结构体声明了 `json:"name"`，v1 也能读入 `{"Name":"Remi"}`。v2 默认按 `name` 精确匹配，`Name` 会被当成未知字段跳过。于是 `Unmarshal` 返回 `nil`，一个刚创建的 `Pet` 里却只剩下空字符串。排查这类问题要看字段值，光断言 `err == nil` 没用。确认有旧客户端依赖这种写法，可以只在对应字段上加 `case:ignore`。 

重复键和非法 UTF-8 倒是容易发现，v2 默认会报错。v1 能读入 `{"name":"Remi","name":"Oliver"}`，最后 `Name` 是 `"Oliver"`。它也会把字符串里的非法 UTF-8 字节替换成 `�`。如果迁移期间还得读一批带重复键的历史数据，可以在那个读取入口传 `jsontext.AllowDuplicateNames(true)`。 

还有一类错误跟输入无关。比如一个结构体字段写了 `json:"owner,string"`，v1 会忽略这里的 `string`，v2 编码时会返回错误。这种标签留着也没有作用，删掉就行。 

## 如何迁移？

如果项目已经有统一的 JSON 封装，可以先在封装里换成 v2，把 v1 的默认选项一起带过去。`jsonv1` 和 `jsonv2` 分别是 `encoding/json` 和 `encoding/json/v2` 的导入别名。

```go
b, err := jsonv2.Marshal(v, jsonv1.DefaultOptionsV1())
```

选项按传入顺序覆盖。比如已经确认某个接口可以把 `null` 切片改成 `[]`，就在 `DefaultOptionsV1()` 后面加 `jsonv2.FormatNilSliceAsNull(false)`。这样一次只动一处行为，出了问题也知道该查哪条接口。

有些接口的调用方太多，单靠测试很难知道线上还有谁在传 `Name`。官方迁移指南提到的 `jsonsplit` 可以用来抽样比对。把调用设为 `CallBothButReturnV1`，同一份数据走两套实现，业务仍然拿到 v1 的结果。双跑有额外开销，高频接口可以先从小比例采样开始。 

最后还是回到接口本身。`null` 变成 `[]`，可能只是少写一个前端判断。`Name` 读进去变成空字符串，可能要等到写库才被发现。比对结果里两条都叫「有差异」，处理方式却完全不同。