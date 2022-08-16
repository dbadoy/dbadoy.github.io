---
layout: post
title:  "os.Stat() vs os.Open() in check the file does exist"
tags: go-benchmark
---

# os.Stat() vs os.Open()
일단, 위의 두 메서드는 사용처가 다르다. os.Stat()의 경우 파일에 대한 정보를 얻을 때 주로 사용하며, os.File()의 경우, 파일을 컨트롤 할 때 사용한다.<br>
내가 이 글에서 비교하고자 하는 상황은, 파일이 존재하는지 확인하는 경우이다.

# 결론
단순히 생각해보았을 때, 파일에 대한 정보만 가져오는 os.Stat()이 훨씬 빠를 것 같다.
실제로도 더 빠르다.

```go
package test

import (
	"os"
	"testing"
)

func BenchmarkStatExistFile(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_, _ = os.Stat("./source")
	}
}

func BenchmarkOpenExistFile(b *testing.B) {

	for i := 0; i < b.N; i++ {
		_, _ = os.Open("./source")

	}
}

func BenchmarkStatNotExistFile(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_, _ = os.Stat("./sourceaaaaaa")
	}
}

func BenchmarkOpenNotExistFile(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_, _ = os.Open("./sourceaaaaaa")
	}
}
```

```
BenchmarkStatExistFile-10                1000000               480.1 ns/op
BenchmarkOpenExistFile-10                1000000               567.5 ns/op
BenchmarkStatNotExistFile-10             1000000               483.3 ns/op
BenchmarkOpenNotExistFile-10             1000000               570.2 ns/op
```

os.Stat()이 더 빠르긴 하지만, 드라마틱하게 빠르진 않다. 약 18% 정도? <br>

막연히 os.Stat()이 빠르다고 생각하던 이유는, os.File()은 직접 파일하고 연결되기 때문에, 파일 정보만 읽어오는 os.Stat()이 더 빠르겠다는 단순한 생각이었다. <br>
하지만, os.File() 리턴값 자체는 파일에 접근할 File Descriptor만 넘어오기 때문에 그렇게 차이가 크지 않았던 것이다. <br><br>

```go
type FileInfo interface {
	Name() string       // base name of the file
	Size() int64        // length in bytes for regular files; system-dependent for others
	Mode() FileMode     // file mode bits
	ModTime() time.Time // modification time
	IsDir() bool        // abbreviation for Mode().IsDir()
	Sys() any           // underlying data source (can return nil)
}
```

benchmark 메모리 할당값
```
BenchmarkStatExistFile-10                1000000               481.4 ns/op           272 B/op          3 allocs/op
BenchmarkOpenExistFile-10                1000000               554.6 ns/op            64 B/op          2 allocs/op
BenchmarkStatNotExistFile-10             1000000               509.7 ns/op           272 B/op          3 allocs/op
BenchmarkOpenNotExistFile-10             1000000               565.8 ns/op            64 B/op          2 allocs/op
```
그러한 점에서, 파일 정보를 변수에 담아 리턴하는 os.Stat()이 메모리 할당량이 더 크다 <br>

<br>
아무튼... 결과적으로,

```go
var err error
// case 1
_, err = os.File("./test.txt")
if os.IsNotExist(err) {
	// not exist
}

// case2
_, err = os.Stat("./test.txt")
if os.IsNotExist(err) {
	// not exist
}
```
위 두개는 굳이 성능을 생각하여 골라 쓸 필요는 없다고 생각된다. 굳이 굳이 성능적인 측면을 비교 해 보자면, 속도가 중요하면 os.Stat(), 메모리 절약이 중요하면 os.File()이 되겠다.