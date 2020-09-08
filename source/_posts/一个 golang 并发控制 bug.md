---
title: 一个 golang 并发控制 bug
date: 2020-09-08 11:20:02
thumbnail: /img/go_channel.jpg
tags: 
- golang
categories:
- IT 技术
toc: true
---
本文翻译自[这里](https://utcc.utoronto.ca/~cks/space/blog/programming/GoConcurrencyStillNotEasy)。

在 golang 语言编程中，由于语言本身就已经实现和支持了并发，实现并发变得很容易。只需通过 go 关键字来开启 goroutine 进行并发操作，并且通过 channel 和 sync.WaitGroup 来进行并发控制。
由于 golang 没有提供并发操作的标准库，并发控制还是需要程序员自己实现，这样经常会出现意想不到的 bug。

<!-- more -->

# 一个不起眼的小 bug

[Gops](https://github.com/google/gops) 是一个用于列出和诊断分析系统中正在运行的 Go 程序的命令行工具。

Gops 在 [goprocess.FindAll()](https://github.com/google/gops/blob/6fb0d860e5fa50629405d9e77e255cd32795967e/goprocess/gp.go#L29) 中实现并发限制，下面是简化代码。

```golang
func FindAll() []P {
   pss, err := ps.Processes()
   [...]
   found := make(chan P)                                // 无缓冲 channel ，用来收集结果
   limitCh := make(chan struct{}, concurrencyProcesses) // 有缓冲 channel ，用来控制并发 

   for _, pr := range pss {
      limitCh <- struct{}{}                             // 占用一个并发名额
      pr := pr
      go func() {
         defer func() { <-limitCh }()                   // 释放一个并发名额
         [... get a P with some error checking ...]
         found <- P                                     // 结果丢到收集结果 channel 中
      }()
   }
   [...]

   var results []P
   for p := range found {                               // 处理结果
      results = append(results, p)
   }
   return results
}
```

上面的代码思路很清晰，是一个比较常见的并发控制模式（详细参见 Go 101's [Channel Use Cases](https://go101.org/article/channel-use-cases.html)）。

上面的代码在第 5 行，`limitCh := make(chan struct{}, concurrencyProcesses)` 生成了带有 concurrencyProcesses 长度缓冲的 channel。
并且在第 8 行，每次 for 循环启动当时候，使用 `limitCh <- struct{}{}` 占用一个并发名额，然后启动协程。
当有 concurrencyProcesses 个协程启动后，就会占满 limitCh，for 循环就会阻塞（也不会有新的协程启动），直至已经启动某个协程处理完，
在第 11 行，使用 `defer func() { <-limitCh }() ` 释放并发名额，才会有新的协程启动。
这样就保障了始终只有 concurrencyProcesses 个协程在运行，达到了控制并发的目的。

乍看之下，这个程序似乎没什么问题。然而这个程序里隐藏里一个不明显的 bug [issue#123](https://github.com/google/gops/issues/123)。
这个 bug 的起因是，程序会先跑完 for 循环，再跑下面第 19 行的处理结果逻辑。然而，之后如果没有进行到第 19 行的话，found channel 会在每次写就阻塞住一个协程。如果有 concurrencyProcesses 个协程阻塞在第 13 行，那么 for 循环将一直阻塞，主协程也运行不到第 19 行，这样就形成了死锁。

# 解决方案

这个 bug 有多种改进方案，

1. 在协程中 执行 limitCh <- struct{}{}。这样虽然主协程会启动很多 子协程，然而大多数协程会阻塞在 `limitCh <- struct{}{}`，每当有协程退出，就会有新的协程进入处理流程。这样也算是一种控制并发。(todo: 搞清楚这样有什么坏处)

```golang
func FindAll() []P {
   [...]
   for _, pr := range pss {
      [...]
      go func() {
         limitCh <- struct{}{}                          // 占用一个并发名额
         defer func() { <-limitCh }()                   // 释放一个并发名额
         [...]
      }()
   }
   [...]
}
```

2. 在 found <- P 前释放 limitCh，（不推荐这么做，因为在 defer 里进行处理会更简单、可靠，否则需要在每个err处理里写一次释放操作）。

```golang
func FindAll() []P {
   [...]
   for _, pr := range pss {
      limitCh <- struct{}{}                             // 占用一个并发名额 
      [...]
      go func() {        
         [... get a P with some error checking ...]
         if err != nil {
            <-limitCh                                   // 释放一个并发名额
            return
         }
         <-limitCh                                      // 释放一个并发名额
         found <- P                                     // 结果丢到收集结果 channel 中
      }()
   }
   [...]
}
```

3. 可以把整个 for 循环放到一个额外的协程中，然后 for 循环即使阻塞掉也不会影响到主协程的继续执行。

```golang
func FindAll() []P {
   [...]
   go func(){
      for _, pr := range pss {
         [...]
      }
      [...]
   }()
}
```

# 总结

通过以上这个案例，咱们可以看出 golang 中的并发控制并不是想象中的那么容易，一个小错误就可以导致程序运行不稳定，造成被阻塞住的情况。
