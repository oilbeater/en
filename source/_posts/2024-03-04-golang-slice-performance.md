---
title: The Impact of Pre-allocating Slice Memory on Performance in Golang
date: 2024-03-04 10:48:09
tags: [performance, golang]
---

- [Theoretical Basis of Slice Memory Allocation](#theoretical-basis-of-slice-memory-allocation)
- [Quantitative Measurement](#quantitative-measurement)
- [Lint Tool prealloc](#lint-tool-prealloc)
- [Summary](#summary)

During my code reviews, I often focus on whether the slice initialization in the code has allocated the expected memory space, that is, I always request to change from `var init []int64` to `init := make([]int64, 0, length)` format whenever possible. However, I had no quantitative concept of how much this improvement affects performance, and it was more of a dogmatic requirement. This blog will introduce the theoretical basis of how pre-allocating memory improves performance, quantitative measurements, and tools for automated detection.

# Theoretical Basis of Slice Memory Allocation

The implementation for Golang Slice expansion can be found in [slice.go under growslice](https://github.com/golang/go/blob/go1.20.6/src/runtime/slice.go#L157). The general idea is that when the Slice capacity is less than 256, each expansion will create a new slice with double the capacity; when the capacity exceeds 256, each expansion will create a new slice with 1.25 times the original capacity. Afterwards, the old slice's data is copied to the new slice, ultimately returning the new slice.

The expansion code is as follows:
```go
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		newcap = newLen
	} else {
		const threshold = 256
		if oldCap < threshold {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < newLen {
				// Transition from growing 2x for small slices
				// to growing 1.25x for large slices. This formula
				// gives a smooth-ish transition between the two.
				newcap += (newcap + 3*threshold) / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = newLen
			}
		}
	}
```

Theoretically, if the slice's capacity is pre-allocated, eliminating the need for dynamic expansion, we can see performance improvements in several areas:

1. Memory needs to be allocated only once, avoiding repeated allocations.
2. Data copying is not required repeatedly.
3. There's no need for repeated garbage collection of the old slice.
4. Memory is allocated accurately, avoiding the capacity waste caused by dynamic allocation.

In theory, pre-allocating slice capacity should lead to performance improvements compared to dynamic allocation, but the exact amount of improvement requires quantitative measurement.

# Quantitative Measurement

We refer to the code from [prealloc](https://github.com/alexkohler/prealloc/blob/master/prealloc_test.go) and make simple modifications to measure the impact of pre-allocating vs. dynamically allocating slices of different capacities on performance.

The test code is as follows, and by changing `length`, we can observe performance data under different scenarios:

```go title="prealloc_test.go"
package prealloc_test

import "testing"

var length = 1000

func BenchmarkNoPreallocate(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		// Don't preallocate our initial slice
		var init []int64
		for j := 0; j < length; j++ {
			init = append(init, 0)
		}
	}
}

func BenchmarkPreallocate(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		// Preallocate our initial slice
		init := make([]int64, 0, length)
		for j := 0; j < length; j++ {
			init = append(init, 0)
		}
	}
}
```

The first function tests the performance of dynamic allocation, and the second tests the performance of pre-allocation. The tests can be executed with the following command:

```bash
go test -bench=. -benchmem prealloc_test.go
```

Results with `length = 1`:

```bash
BenchmarkNoPreallocate-12       40228154                27.36 ns/op            8 B/op          1 allocs/op
BenchmarkPreallocate-12         55662463                19.97 ns/op           

 8 B/op          1 allocs/op
```

With `length = 1`, theoretically, both dynamic and static allocation should perform a single initial memory allocation, so there should be no difference in performance. However, pre-allocation takes 70% of the time compared to dynamic allocation, showing a 1.4x performance advantage even when the number of memory allocations remains the same. This performance improvement seems related to the continuity of variable allocation.

Results with `length = 10`:

```bash
BenchmarkNoPreallocate-12        5402014               228.3 ns/op           248 B/op          5 allocs/op
BenchmarkPreallocate-12         21908133                50.46 ns/op           80 B/op          1 allocs/op
```

With `length = 10`, pre-allocation still only performs one memory allocation, while dynamic allocation performs 5, making pre-allocation's performance 4 times better. This indicates that even for smaller slice sizes, pre-allocation can significantly improve performance.

Results with `length` at 129, 1025, and 10000:

```bash
# length = 129
BenchmarkNoPreallocate-12         743293              1393 ns/op            4088 B/op          9 allocs/op
BenchmarkPreallocate-12          3124831               386.1 ns/op          1152 B/op          1 allocs/op

# length = 1025
BenchmarkNoPreallocate-12         169700              6571 ns/op           25208 B/op         12 allocs/op
BenchmarkPreallocate-12           468880              2495 ns/op            9472 B/op          1 allocs/op

# length = 10000
BenchmarkNoPreallocate-12          14430             86427 ns/op          357625 B/op         19 allocs/op
BenchmarkPreallocate-12            56220             20693 ns/op           81920 B/op          1 allocs/op
```

At larger capacities, static allocation still only requires one memory allocation, but the performance improvement does not scale proportionally, typically 2 to 4 times better. This may be due to other overheads or special optimizations in Golang for large capacity copying, so the performance gap does not widen as much.

When changing the slice's contents to a more complex struct, it was expected that copying would incur greater performance overhead, but in practice, the performance gap between pre-allocation and dynamic allocation for complex structs was even smaller. It appears there are many internal optimizations, and the behavior does not always align with intuition.

# Lint Tool prealloc

Although pre-allocating memory can bring certain performance improvements, relying solely on manual review for this issue in larger projects is prone to oversights. This is where lint tools for automatic code scanning become necessary. [prealloc](https://github.com/alexkohler/prealloc) is such a tool that can scan for potential slices that could be pre-allocated but were not, and it can be integrated into [golangci-lint](https://golangci-lint.run/usage/linters/#prealloc).

# Summary

Overall, pre-allocating slice memory is a relatively simple yet effective optimization method. Even when slice capacities are small, pre-allocation can still significantly improve performance. Using static code scanning tools like prealloc, these potential optimizations can be easily detected and integrated into CI, simplifying future operations.

> Update: I previously wrote about [The Impact of Preallocating Slice Memory in Golang](https://oilbeater.com/en/2024/03/04/golang-slice-performance/), discussing the performance effects of preallocating memory in Slices. The scenarios considered were relatively simple, and recently, I conducted further tests to provide more information, including the impact of appending an entire Slice and the use of sync.Pool and bytebufferpool on performance here [The Impact of Preallocating Slice Memory in Golang (Continued)](https://oilbeater.com/en/2024/03/04/golang-slice-performance-cont/)