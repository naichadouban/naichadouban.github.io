---
title: colly爬虫队列queue源码分析
date: 2019-06-02
tags: ["golang","爬虫"]
categories: ["golang","apider"]
---

# queue
先来看最简单的队列使用
```
func main() {
	url := "https://httpbin.org/delay/1"

	// Instantiate default collector
	c := colly.NewCollector()

	// create a request queue with 2 consumer threads
	q, _ := queue.New(
		2, // Number of consumer threads
		&queue.InMemoryQueueStorage{MaxSize: 10000}, // Use default queue storage
	)

	c.OnRequest(func(r *colly.Request) {
		fmt.Println("visiting", r.URL)
	})

	for i := 0; i < 5; i++ {
		// Add URLs to the queue
		q.AddURL(fmt.Sprintf("%s?n=%d", url, i))
	}
	// Consume URLs
	q.Run(c)
}
```
## 创建队列
```
q, _ := queue.New(
		2, // Number of consumer threads
		&queue.InMemoryQueueStorage{MaxSize: 10000}, // Use default queue storage
	)
```
queue.New()生成一个新的队列，传入两个参数，一个是协程数，一个是存储（InMemoryQueueStorage），存储的对象是`Request`对象。InMemoryQueueStorage只是`Storage`接口的一个实现,将内容存储在内存中的实现。
```
// Storage is the interface of the queue's storage backend
type Storage interface {
	// Init initializes the storage
	Init() error
	// AddRequest adds a serialized request to the queue
	AddRequest([]byte) error
	// GetRequest pops the next request from the queue
	// or returns error if the queue is empty
	GetRequest() ([]byte, error)
	// QueueSize returns with the size of the queue
	QueueSize() (int, error)
}
```
这句代码之后，我们就有了两个结构体对象
```
// InMemoryQueueStorage is the default implementation of the Storage interface.
// InMemoryQueueStorage holds the request queue in memory.
type InMemoryQueueStorage struct {
	// MaxSize defines the capacity of the queue.
	// New requests are discarded if the queue size reaches MaxSize
	MaxSize int
	lock    *sync.RWMutex
	size    int
	first   *inMemoryQueueItem
	last    *inMemoryQueueItem
}
```
```
// Queue is a request queue which uses a Collector to consume
// requests in multiple threads
type Queue struct {
	// Threads defines the number of consumer threads
	Threads           int
	storage           Storage
	activeThreadCount int32
	threadChans       []chan bool
	lock              *sync.Mutex
}
```
## 往队列中添加请求
```
for i := 0; i < 5; i++ {
		// Add URLs to the queue
		q.AddURL(fmt.Sprintf("%s?n=%d", url, i))
	}
```
我们看AddURL()方法。
```
// AddURL adds a new URL to the queue
func (q *Queue) AddURL(URL string) error {
	u, err := url.Parse(URL)
	if err != nil {
		return err
	}
	r := &colly.Request{
		URL:    u,
		Method: "GET",
	}
	d, err := r.Marshal()
	if err != nil {
		return err
	}
	return q.storage.AddRequest(d)
}
```
`r.Marshal()`代码，就是把`Request`序列化。`r.Marshal()`部分代码
```
sr := &serializableRequest{
		URL:    r.URL.String(),
		Method: r.Method,
		Body:   body,
		ID:     r.ID,
		Ctx:    ctx,
	}
if r.Headers != nil {
		sr.Headers = *r.Headers
	}
	return json.Marshal(sr)
```
然后` return q.storage.AddRequest(d)`,就是把`Request`序列化得到的数据添加到队列的存储中。可以看到`inMemoryQueueItem`结构体中，有两个字段`first`,`last`,这两个字段共同决定了一个链表。
具体看下
```
// AddRequest implements Storage.AddRequest() function
func (q *InMemoryQueueStorage) AddRequest(r []byte) error {
	.....
	i := &inMemoryQueueItem{Request: r}
	if q.first == nil {
		q.first = i
	} else {
		q.last.Next = i
	}
	q.last = i
	q.size++
	return nil
}
```
1. `q.first==nil`的情况，both `first`  and `last` refer to new object
2. `q.first!=nil`的时候，first不用动，`q.last.Next=i`,然后`q.last=i`

# 队列启动Run()

```
// Run starts consumer threads and calls the Collector
// to perform requests.
// Run blocks while the queue has active requests. 这句话什么意思？
func (q *Queue) Run(c *colly.Collector) error {
	wg := &sync.WaitGroup{}
	for i := 0; i < q.Threads; i++ {
		wg.Add(1)
		go func(c *colly.Collector, wg *sync.WaitGroup) {
			defer wg.Done()
			for {
				// IsEmpty()根据q.Size判断
				if q.IsEmpty() {
					// queue是空的，并且恰好激活状态协程也是0。就直接break
					if q.activeThreadCount == 0 {
						break
					}
					ch := make(chan bool)
					q.lock.Lock()
					q.threadChans = append(q.threadChans, ch)
					q.lock.Unlock()
					// TODO 这里怎么会收到信息？
					action := <-ch
					if action == stop && q.IsEmpty() {
						break
					}
				}
				q.lock.Lock()
				atomic.AddInt32(&q.activeThreadCount, 1)
				q.lock.Unlock()
				// 如何存储的队列中没有request了，那就不往下进行了
				rb, err := q.storage.GetRequest()
				if err != nil || rb == nil {
					q.finish()
					continue
				}
				// note: collector在这一句代码中已经被包裹在Request对象中了
				r, err := c.UnmarshalRequest(rb)
				if err != nil || r == nil {
					q.finish()
					continue
				}
				// r.Do()就是真的发送请求了。
				// TODO 感觉还是没有解决并发的问题：
				//  一个visit之后，在OnResponse()中，再次进行visit()->
				// 1. OnHtml()是不是一直不能调用了
				// 2. collector设置的回调函数，只能对最后一次请求起作用了。
				r.Do()
				q.finish()
			}
		}(c, wg)
	}
	wg.Wait()
	return nil
}
```
> TODO 感觉还是没有解决并发的问题：
一个visit之后，在OnResponse()中，再次进行visit()
(1.)  OnHtml()是不是一直不能调用了
(2.) collector设置的回调函数，只能对最后一次请求起作用了。
```
自己想到的解决办法：
就是对不同的url
1. 设置不同的collector对象
2. 要使用队列的话，设置不同的队列queue
```
