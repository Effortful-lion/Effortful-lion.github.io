---
title: "Channel"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: true
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Channel 源码解读

# 扩展：

编写一个高性能的无锁RingBuffer，对比与 Channel 的差异

回答：
```text

```

**编写：**
```go
package ringbuffer

import (
	"fmt"
	"math"
	"strings"
)

// 线程不安全（单线程）的 ringbuffer
// 核心规则：
// 1. 固定大小，满了覆盖最旧数据
// 2. head 指向最旧数据，tail 指向下一个待插入位置
// 3. count 记录当前元素个数
type RingBuffer struct {
	buf   []int
	head  int // 最旧数据的位置
	tail  int // 下一个待插入的位置
	count int // 当前元素个数
	cap   int // 固定容量（语义更清晰）
}

// NewRingBuffer 创建指定容量的ringbuffer
func NewRingBuffer(size int) *RingBuffer {
	if size <= 0 {
		panic("size must be greater than 0") // 防御性检查
	}
	return &RingBuffer{
		buf:   make([]int, size),
		head:  0,
		tail:  0,
		count: 0,
		cap:   size,
	}
}

// Count 返回当前元素个数
func (rb *RingBuffer) Count() int {
	return rb.count
}

// Size 返回ringbuffer的固定容量
func (rb *RingBuffer) Size() int {
	return rb.cap
}

// Push 写入数据，满了覆盖最旧数据
func (rb *RingBuffer) Push(v int) {
	// 写入新数据到tail位置
	rb.buf[rb.tail] = v
	// 满了则移动head（覆盖最旧数据），否则count+1
	if rb.count == rb.cap {
		rb.head = (rb.head + 1) % rb.cap // 覆盖旧数据，head后移
	} else {
		rb.count++ // 没满，计数+1
	}
	// tail永远后移（环形）
	rb.tail = (rb.tail + 1) % rb.cap
}

// Pop 弹出最旧数据，空则返回math.MinInt32
func (rb *RingBuffer) Pop() int {
	if rb.count == 0 {
		return math.MinInt32
	}
	v := rb.buf[rb.head]
	rb.buf[rb.head] = 0 // 可选：清空原位置（便于调试）
	rb.head = (rb.head + 1) % rb.cap
	rb.count--
	return v
}

// String 格式化输出ringbuffer内容（[v1, v2, v3]）
func (rb *RingBuffer) String() string {
	if rb.count == 0 {
		return "[]"
	}
	sb := strings.Builder{}
	sb.WriteString("[")

	count := 0
	for i := rb.head; count < rb.count; i = (i + 1) % rb.cap {
		v := rb.buf[i]
		if count == rb.count-1 {
			sb.WriteString(fmt.Sprintf("%d", v))
		} else {
			sb.WriteString(fmt.Sprintf("%d, ", v))
		}
		count++
	}

	sb.WriteString("]")
	return sb.String()
}
```

**测试：**
```go
package ringbuffer

import (
	"fmt"
	"math"
	"sync"
	"testing"
)

func Test_RB(t *testing.T) {
	rb := NewRingBuffer(5)
	nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}
	for _, num := range nums {
		rb.Push(num)
		fmt.Printf("buffer：%s\n", rb.String())
	}
}

// TestRingBuffer_New 测试创建ringbuffer的边界情况
func TestRingBuffer_New(t *testing.T) {
	// 测试1：创建容量为0的ringbuffer，应panic
	defer func() {
		if r := recover(); r == nil {
			t.Error("创建容量为0的ringbuffer未触发panic")
		}
	}()
	NewRingBuffer(0)

	// 测试2：创建正常容量的ringbuffer
	rb := NewRingBuffer(3)
	if rb.Size() != 3 || rb.Count() != 0 {
		t.Errorf("创建容量为3的ringbuffer失败，期望Size=3/Count=0，实际Size=%d/Count=%d", rb.Size(), rb.Count())
	}
}

// TestRingBuffer_Push 测试写入数据（未覆盖/覆盖场景）
func TestRingBuffer_Push(t *testing.T) {
	rb := NewRingBuffer(3)

	// 场景1：写入未超容量（3个数据）
	rb.Push(1)
	rb.Push(2)
	rb.Push(3)
	if rb.Count() != 3 || rb.String() != "[1, 2, 3]" {
		t.Errorf("写入3个数据失败，期望Count=3/内容[1,2,3]，实际Count=%d/内容%s", rb.Count(), rb.String())
	}

	// 场景2：写入超容量（覆盖最旧数据）
	rb.Push(4)
	if rb.Count() != 3 || rb.String() != "[2, 3, 4]" {
		t.Errorf("覆盖写入失败，期望Count=3/内容[2,3,4]，实际Count=%d/内容%s", rb.Count(), rb.String())
	}

	// 场景3：继续覆盖写入
	rb.Push(5)
	if rb.Count() != 3 || rb.String() != "[3, 4, 5]" {
		t.Errorf("多次覆盖写入失败，期望Count=3/内容[3,4,5]，实际Count=%d/内容%s", rb.Count(), rb.String())
	}
}

// TestRingBuffer_Pop 测试弹出数据（正常弹出/空队列弹出）
func TestRingBuffer_Pop(t *testing.T) {
	rb := NewRingBuffer(3)

	// 场景1：空队列弹出
	v := rb.Pop()
	if v != math.MinInt32 {
		t.Errorf("空队列弹出失败，期望返回%d，实际返回%d", math.MinInt32, v)
	}

	// 场景2：正常弹出（未空）
	rb.Push(1)
	rb.Push(2)
	rb.Push(3)
	v = rb.Pop()
	if v != 1 || rb.Count() != 2 || rb.String() != "[2, 3]" {
		t.Errorf("弹出第一个数据失败，期望值=1/Count=2/内容[2,3]，实际值=%d/Count=%d/内容%s", v, rb.Count(), rb.String())
	}

	// 场景3：弹出至空
	rb.Pop() // 弹出2
	rb.Pop() // 弹出3
	if rb.Count() != 0 || rb.String() != "[]" {
		t.Errorf("弹出至空失败，期望Count=0/内容[]，实际Count=%d/内容%s", rb.Count(), rb.String())
	}

	// 场景4：空队列再次弹出
	v = rb.Pop()
	if v != math.MinInt32 {
		t.Errorf("空队列弹出失败，期望返回%d，实际返回%d", math.MinInt32, v)
	}
}

// TestRingBuffer_Mix 测试混合读写场景（最接近实际使用）
func TestRingBuffer_Mix(t *testing.T) {
	rb := NewRingBuffer(3)

	// 步骤1：写入2个数据
	rb.Push(10)
	rb.Push(20)
	if rb.Count() != 2 || rb.String() != "[10, 20]" {
		t.Error("步骤1失败：", rb.String())
	}

	// 步骤2：弹出1个数据
	rb.Pop()
	if rb.Count() != 1 || rb.String() != "[20]" {
		t.Error("步骤2失败：", rb.String())
	}

	// 步骤3：写入3个数据（覆盖）
	rb.Push(30)
	rb.Push(40)
	rb.Push(50)
	if rb.Count() != 3 || rb.String() != "[30, 40, 50]" {
		t.Error("步骤3失败：", rb.String())
	}

	// 步骤4：弹出2个数据
	rb.Pop()
	rb.Pop()
	if rb.Count() != 1 || rb.String() != "[50]" {
		t.Error("步骤4失败：", rb.String())
	}

	// 步骤5：写入2个数据（未覆盖）
	rb.Push(60)
	rb.Push(70)
	if rb.Count() != 3 || rb.String() != "[50, 60, 70]" {
		t.Error("步骤5失败：", rb.String())
	}
}

// 这个需要稍加改造，加个读写锁就好了
func TestSafeRingBuffer_ConcurrentPushPop(t *testing.T) {
	// 跳过短测试（如需运行，注释此行）
	// t.Skip("高并发压测，按需运行")

	srb := NewSafeRingBuffer(100)
	var wg sync.WaitGroup
	total := 10000 // 总读写次数

	// 1. 启动10个写入goroutine
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < total/10; j++ {
				srb.Push(j)
			}
		}()
	}

	// 2. 启动10个读取goroutine
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			count := 0
			for count < total/10 {
				if ok := srb.Pop(); ok != 0 {
					count++
				}
			}
			t.Log("输出count:", count)
		}()
	}

	// 3. 等待所有goroutine完成
	wg.Wait()

	// 4. 验证无panic、无死锁（核心是程序能正常结束）
	t.Log("高并发读写测试完成，无异常")
}
```