# [实战Go并发：构建一个简易高效的并发网络爬虫](https://keeaiei.github.io/)
Go语言内置的Goroutines和Channels为编写并发程序提供了强大的原语支持。网络爬虫是一个典型的I/O密集型任务，非常适合用Go的并发模型来实现，以充分利用系统资源，显著提升爬取效率。本文将展示如何使用Go（借助第三方库colly）快速构建一个简易但高效的并发网络爬虫。

核心思路：

并发抓取： 利用Goroutines并发地发起多个HTTP请求，避免单个请求的等待阻塞整个程序。

任务调度与协调： 使用Channel来分发待抓取的URL任务，并收集结果或控制并发数（Worker Pool模式）。

优雅停止： 利用context.Context或Channel信号来通知所有Goroutine在完成任务或收到中断时安全退出。

实现步骤 (使用 colly 框架)：

安装依赖：

bash
go get github.com/gocolly/colly/v2
代码实现 (concurrent_crawler.go):

go
package main

import (
    "fmt"
    "log"
    "sync"

    "github.com/gocolly/colly/v2"
)

func main() {
    // 种子URL列表
    startURLs := []string{
        "https://example.com/page1",
        "https://example.com/page2",
        // ... 更多起始点
    }

    // 创建一个Collector实例（代表一个爬虫）
    c := colly.NewCollector(
        // 可选：限制域名
        colly.AllowedDomains("example.com", "www.example.com"),
        // 可选：设置并发请求数 (默认是巨大的，这里限制为5个并行)
        colly.Async(true), // 启用异步
    )
    c.Limit(&colly.LimitRule{DomainGlob: "*", Parallelism: 5}) // 设置全局并发限制

    // 使用WaitGroup等待所有抓取任务完成
    var wg sync.WaitGroup

    // 注册回调：当发现新链接时 (在 `<a href="...">` 上)
    c.OnHTML("a[href]", func(e *colly.HTMLElement) {
        link := e.Request.AbsoluteURL(e.Attr("href")) // 解析为绝对URL
        if link != "" {
            // 将新发现的URL加入队列等待抓取 (colly内部管理队列并发)
            e.Request.Visit(link)
        }
    })

    // 注册回调：每当一个请求完成时（无论成功失败）
    c.OnResponse(func(r *colly.Response) {
        defer wg.Done() // 完成一个任务，WaitGroup计数器减1
        log.Printf("Visited: %s\n", r.Request.URL)
        // 在这里处理抓取到的页面内容 r.Body...
        // 例如：解析数据、存储等
        fmt.Printf("Processed page: %s (Size: %d bytes)\n", r.Request.URL, len(r.Body))
    })

    // 注册回调：发生错误时
    c.OnError(func(r *colly.Response, err error) {
        defer wg.Done() // 即使出错，也标记任务完成
        log.Printf("Error visiting %s: %v\n", r.Request.URL, err)
    })

    // 遍历所有种子URL，启动抓取
    for _, url := range startURLs {
        wg.Add(1) // 为每个种子URL增加WaitGroup计数器
        // 将URL加入爬虫队列 (colly内部使用Goroutines处理)
        c.Visit(url)
    }

    // 等待所有由Visit发起的抓取任务完成（包括种子URL和后续发现的URL）
    wg.Wait()
    log.Println("Crawling finished!")
}
代码解析：

colly.NewCollector: 创建爬虫核心对象。colly.Async(true)启用异步模式（即并发），colly.Limit设置并发请求数限制（这里是5个并行请求）。

sync.WaitGroup: 用于主Goroutine等待所有抓取任务（每个Visit启动的任务）完成。每个Visit调用前wg.Add(1)，在每个请求完成（OnResponse）或出错（OnError）时调用defer wg.Done()。

OnHTML回调： 使用CSS选择器a[href]查找页面上的所有链接。e.Request.AbsoluteURL将相对URL转换为绝对URL。e.Request.Visit(link)将新URL加入爬虫的内部队列。colly框架内部使用Goroutines和队列来管理这些新URL的并发抓取。

OnResponse回调： 处理成功抓取的页面内容（r.Body）。这里是日志和打印大小，实际应用应替换为数据解析和存储逻辑。

OnError回调： 处理抓取过程中发生的错误。

启动流程： 遍历种子URL，为每个调用c.Visit(url)并wg.Add(1)。c.Visit是非阻塞的，它会将URL放入内部队列。colly的Worker Goroutines会从队列中取出URL并发执行抓取。

wg.Wait(): 主Goroutine在此阻塞，直到所有由Visit启动的任务都调用了wg.Done()（即在OnResponse或OnError中完成），确保程序在所有抓取完成后才退出。

Go并发在此案例中的优势：

高效利用I/O等待时间： 当一个Goroutine在等待网络响应时，其他Goroutine可以继续处理任务或发起新的请求，CPU资源得到充分利用。

轻量级： 创建和管理成千上万个抓取任务（Goroutines）开销极小。

简洁清晰： 使用colly框架和Go的并发原语（WaitGroup），代码逻辑清晰，避免了手动管理线程池和锁的复杂性。colly内部对队列和并发的封装进一步简化了开发。

易于控制并发度： 通过colly.Limit可以方便地控制爬虫的并发请求数，避免对目标服务器造成过大压力或被封禁。

总结：
这个简单的爬虫示例展示了如何利用Go的Goroutines和Channels（通过colly框架间接使用）以及sync.WaitGroup来实现高效的并发网络爬取。Go的并发模型让开发者能够以相对直观的方式编写出高性能、资源利用率高的I/O密集型应用程序。扩展此爬虫的功能（如深度控制、去重、数据存储、代理支持等）都可以在colly的框架和Go强大的生态下方便地实现。
