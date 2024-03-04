---
title: The Impact of Preallocating Slice Memory in Golang (Continued)
date: 2024-03-04 18:28:09
tags: [performance, golang]
---

- [Basic Performance Tests](#basic-performance-tests)
- [Appending an Entire Slice](#appending-an-entire-slice)
- [Reusing Slices](#reusing-slices)
- [sync.Pool](#syncpool)
- [bytebufferpool](#bytebufferpool)
- [Conclusion](#conclusion)

I previously wrote about [The Impact of Preallocating Slice Memory in Golang](https://oilbeater.com/2024/03/04/golang-slice-performance/), discussing the performance effects of preallocating memory in Slices. The scenarios considered were relatively simple, and recently, I conducted further tests to provide more information, including the impact of appending an entire Slice and the use of sync.Pool on performance.

# Basic Performance Tests

The initial Benchmark code only considered whether the Slice was preallocated with space. The specific code is as follows:

```go
package prealloc_test

import (
    "sync"
    "testing"
)

var length = 1024
var testtext = make([]byte, length, length)

func BenchmarkNoPreallocateByElement(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        // Don't preallocate our initial slice
        var init []byte
        for j := 0; j < length; j++ {
            init = append(init, testtext[j])
        }
    }
}

func BenchmarkPreallocateByElement(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        // Preallocate our initial slice
        init := make([]byte, 0, length)
        for j := 0; j < length; j++ {
            init = append(init, testtext[j])
        }
    }
}
```

The test results are as follows:

```bash
BenchmarkNoPreallocateByElement-12        569978              2151 ns/op            3320 B/op          9 allocs/op
BenchmarkPreallocateByElement-12          804807              1304 ns/op            1024 B/op          1 allocs/op
```

It's clear that not preallocating resulted in 8 additional memory allocations, and roughly, 40% of the time was spent on these extra 8 memory allocations.

These two test cases use a loop to append elements one by one, but the performance gap is not evident when appending an entire Slice. Moreover, in these two test cases, we can't determine the proportion of time consumed by memory allocation.

# Appending an Entire Slice

Therefore, two test cases for appending an entire Slice were added to observe whether preallocating memory still has a significant impact on performance. The new case code is as follows:

```go

func BenchmarkNoPreallocate(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        // Don't preallocate our initial slice
        var init []byte
        init = append(init, testtext...)
    }
}

func BenchmarkPreallocate(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        // Preallocate our initial slice
        init := make([]byte, 0, length)
        init = append(init, testtext...)
    }
}
```

The test results are as follows:

```bash
BenchmarkNoPreallocateByElement-12        569978              2151 ns/op            3320 B/op          9 allocs/op
BenchmarkPreallocateByElement-12          804807              1304 ns/op            1024 B/op          1 allocs/op
BenchmarkNoPreallocate-12                3829890               311.5 ns/op          1024 B/op          1 allocs/op
BenchmarkPreallocate-12                  3968048               306.7 ns/op          1024 B/op          1 allocs/op
```

Both cases only involved one memory allocation, with almost identical time consumption, significantly lower than appending elements one by one. On the one hand, appending an entire Slice knows the final size at the time of Slice expansion, eliminating the need for dynamic memory allocation and reducing overhead. On the other hand, appending an entire Slice involves block copying, reducing loop overhead and significantly improving performance.

However, there is still one memory allocation in each case, and we cannot determine the proportion of time it consumes overall.

# Reusing Slices

To calculate the cost of a single memory allocation, we designed a new test case, placing the Slice's creation outside the loop and setting the Slice's length to 0 at the end of

 each loop iteration for reuse. This way, only one memory allocation is made over many tests, which can be considered negligible. The specific code is as follows:

```go
func BenchmarkPreallocate2(b *testing.B) {
    b.ResetTimer()
    init := make([]byte, 0, length)
    for i := 0; i < b.N; i++ {
        // Preallocate our initial slice
        init = append(init, testtext...)
        init = init[:0]
    }
}
```

The test results are as follows:

```bash
BenchmarkNoPreallocateByElement-12        514904              2171 ns/op            3320 B/op          9 allocs/op
BenchmarkPreallocateByElement-12          761772              1333 ns/op            1024 B/op          1 allocs/op
BenchmarkNoPreallocate-12                4041459               320.9 ns/op          1024 B/op          1 allocs/op
BenchmarkPreallocate-12                  3854649               320.1 ns/op          1024 B/op          1 allocs/op
BenchmarkPreallocate2-12                63147178                18.63 ns/op            0 B/op          0 allocs/op
```

This time, the test showed no memory allocation, and the overall time consumed dropped to 5% of the previous cases. Thus, we can roughly calculate that each memory allocation consumed 95% of the time in the previous test cases, a staggering proportion. Therefore, for performance-sensitive scenarios, it's essential to reuse objects as much as possible to avoid the overhead of repeated object creation.

# sync.Pool

In simple scenarios, one can manually clear the Slice and reuse it within the loop, as in the previous test case. However, in real scenarios, object creation often occurs in various parts of the code, necessitating unified management and reuse. Golang's `sync.Pool` serves this purpose, and its use is quite simple. However, its internal implementation is complex, with a lot of lock-free design for performance. For more details on its implementation, see [https://www.cyhone.com/articles/think-in-sync-pool/](https://www.cyhone.com/articles/think-in-sync-pool/).

The redesigned test case using `sync.Pool` is as follows:

```go
var sPool = &sync.Pool{
    New: function() interface{} {
        return make([]byte, 0, length)
    },
}

function BenchmarkPool(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        // Don't preallocate our initial slice
        buf := sPool.Get().([]byte)
        buf = append(buf, testtext...)
        buf = buf[:0]
        sPool.Put(buf)
    }
}
```

Here, `New` provides a constructor function for `sync.Pool` to create an object when none is available. To use it, the `Get` method retrieves an object from the Pool, and after use, the `Put` method returns the object to `sync.Pool`. It's important to manage the object's lifecycle and clear it before returning it to `sync.Pool` to avoid dirty data. The test results are as follows:

```bash
BenchmarkNoPreallocateByElement-12        522565              2129 ns/op            3320 B/op          9 allocs/op
BenchmarkPreallocateByElement-12          781638              1311 ns/op            1024 B/op          1 allocs/op
BenchmarkPoolByElement-12                 957424              1233 ns/op              24 B/op          1 allocs/op
BenchmarkNoPreallocate-12                4057801               310.3 ns/op          1024 B/op          1 allocs/op
BenchmarkPreallocate-12                  3841848               315.4 ns/op          1024 B/op          1 allocs/op
BenchmarkPreallocate2-12                63356907                18.76 ns/op            0 B/op          0 allocs/op
BenchmarkPool-12                        13784712                85.19 ns/op           24 B/op          1 allocs/op
```

As seen, `sync.Pool` still has a small amount of memory allocation, and its performance cost is higher than manually reusing Slices. However, considering its convenience and the significant performance improvement over not using it, it's still a good solution.

However, directly using `sync.Pool` also has two issues:

1. For Slices, the initial memory allocated by `New` is fixed. If the runtime usage exceeds this, there may still be a lot of dynamic memory allocation adjustments.
2. At the other extreme, if a Slice is dynamically expanded to a large size and then returned to `sync.Pool`,

 it may cause memory leaks and waste.

# bytebufferpool

To achieve better runtime performance, [bytebufferpool](https://github.com/valyala/bytebufferpool) builds on `sync.Pool` with some simple statistical rules to minimize the impact of the two issues mentioned above during runtime. (The project's author is Russian, with other projects like fasthttp, quicktemplate, and VictoriaMetrics under his belt, all of which are excellent examples of performance optimization. The Russian tech community often undertakes projects that push performance to its limits.

The main structure in the code is as follows:

```go
// bytebufferpool/pool.go
const (
  minBitSize = 6 // 2**6=64 is a CPU cache line size
  steps      = 20

  minSize = 1 << minBitSize
  maxSize = 1 << (minBitSize + steps - 1)

  calibrateCallsThreshold = 42000
  maxPercentile           = 0.95
)

type Pool struct {
  calls       [steps]uint64
  calibrating uint64

  defaultSize uint64
  maxSize     uint64

  pool sync.Pool
}
```

`defaultSize` is used for the initial size allocation for Slices, and `maxSize` is for rejecting Slices exceeding this size when returned to `sync.Pool`. The core algorithm dynamically adjusts `defaultSize` and `maxSize` based on statistical size information of Slices used at runtime, avoiding extra memory allocation while preventing memory leaks.

This dynamic statistical process is relatively simple, dividing the size of Slices returned to the Pool into 20 intervals for statistics. After `calibrating` calls, it sorts by size, choosing the most frequently used interval size as `defaultSize`. This statistical method avoids many extra memory allocations. Then, by sorting by size and setting the 95% percentile size as `maxSize`, it prevents large objects from entering the Pool statistically. This dynamic adjustment of these two values achieves better performance at runtime.

# Conclusion

- Always specify the capacity when initializing Slices.
- Avoid initializing Slices in loops.
- Consider using sync.Pool for performance-sensitive paths.
- The cost of memory allocation can far exceed that of business logic.
- For reusing byte buffers, consider bytebufferpool.