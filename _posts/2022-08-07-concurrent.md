---
layout: post
title:  "sync.Mutex() vs sync.Atomic()"
tags: go-benchmark
---

# sync.Mutex() vs sync.Atomic()
고루틴을 사용할 때는 특정 변수에 여러 고루틴이 접근하며 데드락이 발생하는 것을 막을 필요가 있다.
해당 글에서는 Mutext와 Atomic의 속도를 비교해 본다.

# 결론
Atomic이 훨씬 빠르다.

```go
package test

import (
	"sync"
	"sync/atomic"
	"testing"
)

type Test struct {
	data uint32
	mu   sync.Mutex
}

// Set
func BenchmarkMutexSet(b *testing.B) {
	t := Test{}
	for i := 0; i < b.N; i++ {
		t.mu.Lock()
		t.data = uint32(i)
		t.mu.Unlock()
	}
}

func BenchmarkAtomicSet(b *testing.B) {
	t := Test{}
	for i := 0; i < b.N; i++ {
		atomic.StoreUint32(&t.data, uint32(i))
	}
}

// Load
func BenchmarkMutexLoad(b *testing.B) {
	t := Test{
		data: 1,
	}
	for i := 0; i < b.N; i++ {
		t.mu.Lock()
		_ = t.data
		t.mu.Unlock()
	}
}

func BenchmarkAtomicLoad(b *testing.B) {
	t := Test{
		data: 1,
	}
	for i := 0; i < b.N; i++ {
		_ = atomic.LoadUint32(&t.data)
	}
}

```

```
# Set
BenchmarkMutexSet-10             1000000                26.86 ns/op
BenchmarkAtomicSet-10            1000000                 0.4546 ns/op
# Load
BenchmarkMutexLoad-10            1000000                18.83 ns/op
BenchmarkAtomicLoad-10           1000000                 0.6662 ns/op
```

물론 사용처가 다르고, 활용 범위가 넓은 Mutex를 주로 사용하게 되긴 하지만... 

```go
type Temp struct {
    .
    .
    .
    mu     sync.Mutex
    isInit bool
}

func (t *Temp) isInit() bool {
    t.mu.Lock()
    defer t.mu.Unlock()
    return t.isInit
}
```
이와 같이 특정 구조체가 초기화 되었는지 확인하기 위한 필드가 있다면 위와 같이 Mutex를 사용하기 보다는,

```go
type Temp struct {
    .
    .
    .
    // 0 == need initial
    // 1 == already initialized
    isInit uint32
}

func (t *Temp) isInit() uint32 {
    return atomic.LoadUint32(&t.inInit)
}

//
{
    if t.isInit() == 1 {
        // already initialized...
    }

}
```
이렇게 Atomic을 쓰는 것도 한 방법이다.