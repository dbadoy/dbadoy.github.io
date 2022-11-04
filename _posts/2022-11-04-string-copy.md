---
layout: post
title:  "bytes.Buffer vs copy string"
tags: go-benchmark
---

# bytes.Buffer vs copy string
String을 byte array에 이어 붙어야 하는 경우가 종종 있다. <br>
여기서는 bytes.Buffer.WriteString()을 쓰는 경우와 copy() 를 쓰는 경우를 비교해본다 <br>

# 결론
copy()가 빠르다.

```go
package test

import (
	"bytes"
	"testing"
)

func BenchmarkBuffer(b *testing.B) {
	for i := 0; i < b.N; i++ {
		buf := bytes.NewBufferString("")
		buf.WriteString("hello,")
		buf.WriteString(" world")
	}
}

func BenchmarkCopy(b *testing.B) {
	for i := 0; i < b.N; i++ {
		bb := make([]byte, 12)
		ref := 0
		ref += copy(bb[:], "hello,")
		ref += copy(bb[ref:], " world")
	}
}
```

```
BenchmarkBuffer-10      10000000                 9.441 ns/op           0 B/op          0 allocs/op
BenchmarkCopy-10        10000000                 1.063 ns/op           0 B/op          0 allocs/op
```

bytes.Buffer.WriteString()의 내부 동작에서 결국에는 copy(b[:], "string") 과 같이 copy를 쓰긴한다. 그러나 Buffer 구조체 안 []byte의 용량을 재할당하는 작업이 포함되어 있기 때문에, 만약 추가할 문자열의 length를 초기에 파악할 수 있다면 바로 copy를 쓰는 것이 더 빠르다. <br>
근데 대부분... 문자열의 length를 buffer 선언 시점에 알 수 있는 경우는 드문것 같다. <br>

```go
buf := bytes.NewBufferString("")
for _, s := range list {
	buf.WriteString(s)
}
.
.
.
```

이런식으로 순회를하며 전개하는 경우라면, 총 추가되어야 할 문자열의 크기를 알지 못하기 때문에 buffer를 사용하여 용량 재할당까지 의존하는 것이 편할 것이다. <br>
또, copy를 쓰겠다고 <br>

```go
totalCap := 0
for _, s := range list {
	totalCap += len(s)
}
buf := make([]byte, totalCap)
ref := 0
for _, s := range list {
	ref += copy(buf[ref:], s)
}
```

문자열 크기를 구하는 for 문을 추가하는 것은 가독성 측면에서 그다지 좋은 선택은 아닌 것 같다. <br>
그러나 속도나 메모리 할당을 신경쓴다면 위와 같은 구현이 말이 안되는 것은 아니다. 500개의 문자열을 갖는 배열으로 테스트를 해봤을 때, 위 코드의 구현 방식이 2배 이상 빨랐으며 메모리 할당이 현저히 적었다(Buffer.WriteString()가 작동할 때, 내부 버퍼의 용량이 꽉차면 크기를 늘린 버퍼를 재할당하는 과정이 있기 때문에). <br>
<br>

다른 글들의 마무리와 마찬가지로... 상황에 맞게 잘 판단하여 사용하자.