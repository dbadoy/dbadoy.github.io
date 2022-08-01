---
layout: post
title:  "errors.New() vs fmt.Errorf()"
tags: go-benchmark
---

# errors.New() vs fmt.Errorf()
에러를 리턴해줄 때 자주 사용하는 두 방법 중 무엇이 더 빠를까?

## 결론
당연히 errors.New()가 빠르다. 그러나 얼마만큼의 차이가 나는 지 테스트 하는 것에 의미를 둔다.


```go
package test

import (
	"errors"
	"fmt"
	"strconv"
	"testing"
)

func BenchmarkNewError(b *testing.B) {
	var err error
	for i := 0; i < b.N; i++ {
		mesg := "error messages " + strconv.Itoa(i) + " generated."
		err = errors.New(mesg)
		_ = err
	}
}

func BenchmarkFmtError(b *testing.B) {
	var err error
	for i := 0; i < b.N; i++ {
		err = fmt.Errorf("error messages %s generated.", strconv.Itoa(i))
		_ = err
	}
}
```

```
BenchmarkNewError-10             1000000                76.44 ns/op
BenchmarkFmtError-10             1000000               129.4 ns/op
```

차이가 나는 것은 알았지만, 꽤 많이 난다. errors.New가 65% 정도 더 빠르다. <br>
그러나... 에러 메세지 리턴 해주는 것에 성능이 정말 중요할까? 

```go
func Test1(code int) error {
	return errors.New("error code : " + strconv.Itoa(code))
}
func Test2(code int) error {
	return fmt.Errorf("error code : %d", code)
}
```
<br>
나는 fmt.Errorf()가 더 좋은 것 같다.

- golang 기본 패키지 중 net/http 에서는 errors.New()를 사용하긴 하더라.
- go-ethereum 에서는 fmt.Errorf()를 많이 사용한다.


[String Concatenation 관련 글 추천](http://cloudrain21.com/go-how-to-concatenate-strings)

