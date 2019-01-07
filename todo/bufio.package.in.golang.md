# bufio package in Golang

golang 通过 package [bufio](https://golang.org/pkg/bufio/) 来支持 buffered I/O。

熟悉一下: Reader, Writer and Scanner…

## bufio.Writer

对 I/O 的频繁操作会极大的影响性能，每次 写 I/O 都是一次 syscall，因此频繁操作会给 CPU 带来极大的负担。对于磁盘来说，批量写操作有着更好的性能。golang 通过 [bufio.Writer](https://golang.org/pkg/bufio/#Writer) 来合并写操作，避免频繁的写操作。因此在写密集场景下，应该使用 bufio.Writer，而不是 [io.Writer](https://golang.org/pkg/io/#Writer)

bufio.Writer 通过将写内容缓存至 buffer，buffer 满了之后，再进行一次写操作，这样极大的优化性能:

```
producer --> buffer --> io.Writer
```

举个例子，9 次写操作，buffer 是如何工作的:

```
producer         buffer           destination (io.Writer)

   a    ----->   a
   b    ----->   ab
   c    ----->   abc
   d    ----->   abcd
   e    ----->   e      ------>   abcd
   f    ----->   ef               abcd
   g    ----->   efg              abcd
   h    ----->   efgh             abcd
   i    ----->   i      ------>   abcdefgh
```

看份代码来感受一下，如何使用:

```
import (
	"bufio"
	"fmt"
)

type Writer int

func (*Writer) Write(p []byte) (n int, err error) {
	fmt.Println(len(p))
	return len(p), nil
}

func main() {
	fmt.Println("Unbuffered I/O")
	w := new(Writer)
	w.Write([]byte{'a'})
	w.Write([]byte{'b'})
	w.Write([]byte{'c'})
	w.Write([]byte{'d'})
	fmt.Println("Buffered I/O")
	bw := bufio.NewWriterSize(w, 3)
	bw.Write([]byte{'a'})
	bw.Write([]byte{'b'})
	bw.Write([]byte{'c'})
	bw.Write([]byte{'d'})
	err := bw.Flush()
	if err != nil {
		panic(err)
	}
}

```

输出结果:

```
Unbuffered I/O
1
1
1
1
Buffered I/O
3
1
```

上面例子，Unbuffered I/O 会直接调用 syscall 然后写到磁盘 (any destination)，产生了 4 次写操作

buffered I/O 设置缓存区大小为 3，当 buffer 满了之后，才会 flush 调用 syscall 到磁盘，因此最终产生 2 次写操作。

> bufio.Writer 默认是 4096 bytes buffer. 可以通过 `NewWriterSize` 来设置 buffer size

### 具体实现

[源码](https://github.com/golang/go/blob/1e3f563b145ad98d2a5fcd4809e25a6a0bc8f892/src/bufio/bufio.go#L524)

```
type Writer struct {
    err error
    buf []byte
    n   int
    wr  io.Writer
}
```

- buf 变量承载了缓冲区，当 buf 满了，或者调用 Flush 时候，才会输出
- err 是 I/O error，当 wr 产生错误时候设置到 err，并不会终止输出，而是会将 buf 内容尽量输出，发现 err 有错误，才会停止调用 wr Flush 内容
- n 是 buffer 中写指针的位置，`Buffered` 会返回 n 的值，即写指针位置

下面给一份简单说明

```

import (
	"bufio"
	"errors"
	"fmt"
)

type Writer int

func (*Writer) Write(p []byte) (n int, err error) {
	fmt.Printf("Write: %q\n", p)
	return 0, errors.New("boom!")
}

func main() {
	w := new(Writer)
	bw := bufio.NewWriterSize(w, 3)
	bw.Write([]byte{'a'})
	bw.Write([]byte{'b'})
	bw.Write([]byte{'c'})
	bw.Write([]byte{'d'})
	err := bw.Flush()
	fmt.Println(err)
}

Write: "abc"
boom!
```

输出说明，Flush 函数并没产生 2 次写操作，Buffered writer 会在发生第一次错误后，尽量写足够多的数据

### Large writes

当 bufio.Writer 检测到写数据实在是太多，将会直接调用 wr，就是 io.Writer 跳过 buf 直接写，这一点需要注意。

```
package main

import (
	"bufio"
	"fmt"
)

type Writer int

func (*Writer) Write(p []byte) (n int, err error) {
	fmt.Printf("%q\n", p)
	return len(p), nil
}

func main() {
	w := new(Writer)
	bw := bufio.NewWriterSize(w, 3)
	bw.Write([]byte("abcd"))
}

abcd
```

### Reset

bufio.Writer 的核心部分是 Buffer，可以通过 Reset 方法来重用 Buffer

下面给大家看一个系统级的 bug。
```
type Writer1 int
func (*Writer1) Write(p []byte) (n int, err error) {
    fmt.Printf("writer#1: %q\n", p)
    return len(p), nil
}
type Writer2 int
func (*Writer2) Write(p []byte) (n int, err error) {
    fmt.Printf("writer#2: %q\n", p)
    return len(p), nil
}
func main() {
    w1 := new(Writer1)
    bw := bufio.NewWriterSize(w1, 2)
    bw.Write([]byte("ab"))
    bw.Write([]byte("cd"))
    w2 := new(Writer2)
    bw.Reset(w2)
    bw.Write([]byte("ef"))
    bw.Flush()
}
writer#1: "ab"
writer#2: "ef"
```

在调用 Reset 前，我们需要调用 Flush 来把数据刷进 Writer。例子里面，cd 数据丢掉了，就是因为下面一段 [代码](https://github.com/golang/go/blob/1e3f563b145ad98d2a5fcd4809e25a6a0bc8f892/src/bufio/bufio.go#L559)

```
func (b *Writer) Reset(w io.Writer) {
    b.err = nil
    b.n = 0
    b.wr = w
}
```

### Buffer free space

我们可以使用 `Available` 来看 buf 的空余容量

```
package main

import (
	"bufio"
	"fmt"
)

type Writer int

func (*Writer) Write(p []byte) (n int, err error) {
	return len(p), nil
}

func main() {
	w := new(Writer)
	bw := bufio.NewWriterSize(w, 2)
	fmt.Println(bw.Available())
	bw.Write([]byte{'a'})
	fmt.Println(bw.Available())
	bw.Write([]byte{'b'})
	fmt.Println(bw.Available())
	bw.Write([]byte{'c'})
	fmt.Println(bw.Available())
}

2
1
0
1
```

### Write{Byte,Rune,String} Methods

- bw.WriteRune
- bw.WriteByte
- bw.WriteString

```
package main

import (
	"bufio"
	"fmt"
)

type Writer int

func (*Writer) Write(p []byte) (n int, err error) {
	return len(p), nil
}

func main() {
	w := new(Writer)
	bw := bufio.NewWriterSize(w, 10)
	fmt.Println(bw.Buffered())
	bw.WriteByte('a')
	fmt.Println(bw.Buffered())
	bw.WriteRune('ł') // 'ł' occupies 2 bytes
	fmt.Println(bw.Buffered())
	bw.WriteString("aa")
	fmt.Println(bw.Buffered())
}

0
1
3
5
```

### ReadFrom

writer 通过实现 [io.ReaderFrom](https://golang.org/pkg/io/#ReaderFrom) 接口，在 EOF 错误之前，从特定的 Reader 读取数据。

```
type ReaderFrom interface {
        ReadFrom(r Reader) (n int64, err error)
}
```

> [io.Copy](https://golang.org/pkg/io/#Copy) 就使用了 io.ReaderFrom interface.

bufio.Writer 也实现了 io.ReaderFrom 接口，可以读取 io.Reader 所有数据

** 记得调用 Flush**

```
package main

import (
	"bufio"
	"fmt"
	"strings"
)

type Writer int

func (*Writer) Write(p []byte) (n int, err error) {
	fmt.Printf("%q\n", p)
	return len(p), nil
}

func main() {
	s := strings.NewReader("onetwothree")
	w := new(Writer)
	bw := bufio.NewWriterSize(w, 3)
	bw.ReadFrom(s)
	err := bw.Flush()
	if err != nil {
		panic(err)
	}
}

"one"
"two"
"thr"
"ee"
```

## bufio.Reader

从 io.Reader 中读取大量数据，bufio.Reader 能够通过减少读操作提升性能

```
io.Reader --> buffer --> consumer
```

举个例子，从磁盘读取 10 个字母，默认操作会产生 10 次 读操作。
bufio 通过一个 buf，一次读取 4 个，然后 io.Reader 每次读 1 个，这样读操作就减少到 3 次

```
abcd -----> abcd -----> a
            abcd -----> b
            abcd -----> c
            abcd -----> d
efgh -----> efgh -----> e
            efgh -----> f
            efgh -----> g
            efgh -----> h
ijkl -----> ijkl -----> i
            ijkl -----> j
```

### Peek
Peek 从 buf 取前 n 个 bytes

- 如果 buffer 没满，且不足够 n 个，会从 io.Reader 读取足够的数据
- 如果 n 比 buffer 容量还大，返回错误 bufio.ErrBufferFull
- 如果 n 比 io.Reader stream size 要大，返回 EOF

```
s1 := strings.NewReader(strings.Repeat("a", 20))
r := bufio.NewReaderSize(s1, 16)
b, err := r.Peek(3)
if err != nil {
    fmt.Println(err)
}
fmt.Printf("%q\n", b)
b, err = r.Peek(17)
if err != nil {
    fmt.Println(err)
}
s2 := strings.NewReader("aaa")
r.Reset(s2)
b, err = r.Peek(10)
if err != nil {
    fmt.Println(err)
}

"aaa"
bufio: buffer full
EOF
```

>  bufio.Reader 默认 buf size 是 16

通常会用 slice 来承载读取的数据，可能会导致 slice 被其他 buffer 数据覆写，引发问题。

```
s1 := strings.NewReader(strings.Repeat("a", 16) + strings.Repeat("b", 16))
r := bufio.NewReaderSize(s1, 16)
b, _ := r.Peek(3)
fmt.Printf("%q\n", b)
r.Read(make([]byte, 16))
r.Read(make([]byte, 15))
fmt.Printf("%q\n", b)


"aaa"
"bbb"
```

### Reset
类似 bufio.Writer，同样可以复用内部 buffer(io.Reader)

而且这样可以避免不必要的 GC

```
s1 := strings.NewReader("abcd")
r := bufio.NewReader(s1)
b := make([]byte, 3)
_, err := r.Read(b)
if err != nil {
    panic(err)
}
fmt.Printf("%q\n", b)
s2 := strings.NewReader("efgh")
r.Reset(s2)
_, err = r.Read(b)
if err != nil {
    panic(err)
}
fmt.Printf("%q\n", b)

"abc"
"efg"
```

### Discard
Discard 就是直接从 buf 丢弃数据。

```
type R struct{}
func (r *R) Read(p []byte) (n int, err error) {
    fmt.Println("Read")
    copy(p, "abcdefghijklmnop")
    return 16, nil
}
func main() {
    r := new(R)
    br := bufio.NewReaderSize(r, 16)
    buf := make([]byte, 4)
    br.Read(buf)
    fmt.Printf("%q\n", buf)
    br.Discard(4)
    br.Read(buf)
    fmt.Printf("%q\n", buf)
}
Read
"abcd"
"ijkl"
```

如果丢弃数据大于 buf 容量，会读取足够的数据，再进行丢弃，[源码](https://github.com/golang/go/blob/83634e9cf2fb7bf1d45737589291da8bdbee132b/src/bufio/bufio.go#L168)

```
type R struct{}
func (r *R) Read(p []byte) (n int, err error) {
    fmt.Println("Read")
    copy(p, "abcdefghijklmnop")
    return 16, nil
}
func main() {
    r := new(R)
    br := bufio.NewReaderSize(r, 16)
    buf := make([]byte, 4)
    br.Read(buf)
    fmt.Printf("%q\n", buf)
    br.Discard(13)
    fmt.Println("Discard")
    br.Read(buf)
    fmt.Printf("%q\n", buf)
}

Read
"abcd"
Read
Discard
"bcde"
```

### Read

bufio.Reader 为了兼容 [io.Reader](https://golang.org/pkg/io/#Reader) 也实现了统一的接口

```
type Reader interface {
        Read(p []byte) (n int, err error)
}
```

需要注意一种情况，如果 bufio.Reader 内部 buf 存在数据， 无论 p 的大小，都只会从 buf 读取所有数据


```
func (r *R) Read(p []byte) (n int, err error) {
    fmt.Println("Read")
    copy(p, "abcd")
    return 4, nil
}
func main() {
    r := new(R)
    br := bufio.NewReader(r)
    buf := make([]byte, 2)
    n, err := br.Read(buf)
    if err != nil {
        panic(err)
    }
    buf = make([]byte, 4)
    n, err = br.Read(buf)
    if err != nil {
        panic(err)
    }
    fmt.Printf("read = %q, n = %d\n", buf[:n], n)
}

Read
read = "cd", n = 2
```

除非 bufio.Reader 内部 buf 没有数据，才会从内部 io.Reader 读取数据

```
n, err := br.Read(buf)
```

如果 bufio.Reader 内部 buf 没有数据，并且传递的 slice 大于 buf size，将会跳过 buf，直接从内部 io.Reader 读取数据到 slice

```
type R struct{}
func (r *R) Read(p []byte) (n int, err error) {
    fmt.Println("Read")
    copy(p, strings.Repeat("a", len(p)))
    return len(p), nil
}
func main() {
    r := new(R)
    br := bufio.NewReaderSize(r, 16)
    buf := make([]byte, 17)
    n, err := br.Read(buf)
    if err != nil {
        panic(err)
    }
    fmt.Printf("read = %q, n = %d\n", buf[:n], n)
    fmt.Printf("buffered = %d\n", br.Buffered())
}
Read
read = "aaaaaaaaaaaaaaaaa", n = 17
buffered = 0
```

### {Read, Unread}Byte

- br.UnreadByte
- br.ReadByte

```
r := strings.NewReader("abcd")
br := bufio.NewReader(r)
byte, err := br.ReadByte()
if err != nil {
    panic(err)
}
fmt.Printf("%q\n", byte)
fmt.Printf("buffered = %d\n", br.Buffered())
err = br.UnreadByte()
if err != nil {
    panic(err)
}
fmt.Printf("buffered = %d\n", br.Buffered())
byte, err = br.ReadByte()
if err != nil {
    panic(err)
}
fmt.Printf("%q\n", byte)
fmt.Printf("buffered = %d\n", br.Buffered())
'a'
buffered = 3
buffered = 4
'a'
buffered = 3
```

### {Read, Unread}Rune
类似于上面，只不过是专门处理 UTF-8 encoded 字符

### ReadSlice
读取一段字符串，直到出现特定字符

```
func (b *Reader) ReadSlice(delim byte) (line []byte, err error)
```

```
s := strings.NewReader("abcdef|ghij")
r := bufio.NewReader(s)
token, err := r.ReadSlice('|')
if err != nil {
    panic(err)
}
fmt.Printf("Token: %q\n", token)

Token: "abcdef|"
```

如果内容不包含特定的字符，且 buf 没有满，将会返回 io.EOF 错误

```
s := strings.NewReader("abcdefghij")
r := bufio.NewReader(s)
token, err := r.ReadSlice('|')
if err != nil {
    panic(err)
}
fmt.Printf("Token: %q\n", token)

panic: EOF
```

如果 buf 已经填满，还是不包含特定字符，会返回 io.ErrBufferFull 错误

```
s := strings.NewReader(strings.Repeat("a", 16) + "|")
r := bufio.NewReaderSize(s, 16)
token, err := r.ReadSlice('|')
if err != nil {
    panic(err)
}
fmt.Printf("Token: %q\n", token)

panic: bufio: buffer full
```

### ReadBytes

ReadBytes 类似 ReadSlice，但是 ReadBytes 会可以多次调用 ReadSlice，且对 buf size 不敏感

```
func (b *Reader) ReadBytes(delim byte) ([]byte, error)
```

可以看个例子

```
package main

import (
	"bufio"
	"fmt"
	"strings"
)

func main() {
	s := strings.NewReader(strings.Repeat("a", 40) + "|")
	r := bufio.NewReaderSize(s, 16)
	token, err := r.ReadBytes('|')
	if err != nil {
		panic(err)
	}
	fmt.Printf("Token: %q\n", token)
}

Token: "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa|"
```

### ReadString
ReadString 是对 ReadBytes 的简单包装

```
func (b *Reader) ReadString(delim byte) (string, error) {
    bytes, err := b.ReadBytes(delim)
    return string(bytes), err
}
```

### ReadLine

对 ReadSlice('\n') 的包装，但是处理了不同系统下面的换行符

isPrefix 比较有意思的是，特定字符没有匹配到之前，buf 满了，返回 true

```
ReadLine() (line []byte, isPrefix bool, err error)
```

```
s := strings.NewReader(strings.Repeat("a", 20) + "\n" + "b")
r := bufio.NewReaderSize(s, 16)
token, isPrefix, err := r.ReadLine()
if err != nil {
    panic(err)
}
fmt.Printf("Token: %q, prefix: %t\n", token, isPrefix)
token, isPrefix, err = r.ReadLine()
if err != nil {
    panic(err)
}
fmt.Printf("Token: %q, prefix: %t\n", token, isPrefix)
token, isPrefix, err = r.ReadLine()
if err != nil {
    panic(err)
}
fmt.Printf("Token: %q, prefix: %t\n", token, isPrefix)
token, isPrefix, err = r.ReadLine()
if err != nil {
    panic(err)
}

Token: "aaaaaaaaaaaaaaaa", prefix: true
Token: "aaaa", prefix: false
Token: "b", prefix: false
panic: EOF
```

```
s := strings.NewReader("abc")
r := bufio.NewReaderSize(s, 16)
token, isPrefix, err := r.ReadLine()
if err != nil {
    panic(err)
}
fmt.Printf("Token: %q, prefix: %t\n", token, isPrefix)
s = strings.NewReader("abc\n")
r.Reset(s)
token, isPrefix, err = r.ReadLine()
if err != nil {
    panic(err)
}
fmt.Printf("Token: %q, prefix: %t\n", token, isPrefix)
Token: "abc", prefix: false
Token: "abc", prefix: false
```

### WriteTo
bufio.Reader 实现了 io.WriterTo 接口

```
type WriterTo interface {
        WriteTo(w Writer) (n int64, err error)
}
```

看看如何使用

```
type R struct {
    n int
}
func (r *R) Read(p []byte) (n int, err error) {
    fmt.Printf("Read #%d\n", r.n)
    if r.n >= 10 {
         return 0, io.EOF
    }
    copy(p, "abcd")
    r.n += 1
    return 4, nil
}
func main() {
    r := bufio.NewReaderSize(new(R), 16)
    n, err := r.WriteTo(ioutil.Discard)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Written bytes: %d\n", n)
}
```

输出

```
Read #0
Read #1
Read #2
Read #3
Read #4
Read #5
Read #6
Read #7
Read #8
Read #9
Read #10
Written bytes: 40
```

## bufio.ReadWriter
ReadWriter 嵌入了 Reader 和 Writer，同时实现了两者所有接口

```
type ReadWriter struct {
  	*Reader
  	*Writer
}
```

看个例子就懂了

```
s := strings.NewReader("abcd")
br := bufio.NewReader(s)
w := new(bytes.Buffer)
bw := bufio.NewWriter(w)
rw := bufio.NewReadWriter(br, bw)
buf := make([]byte, 2)
_, err := rw.Read(buf)
if err != nil {
    panic(err)
}
fmt.Printf("%q\n", buf)
buf = []byte("efgh")
_, err = rw.Write(buf)
if err != nil {
    panic(err)
}
err = rw.Flush()
if err != nil {
   panic(err)
}
fmt.Println(w.String())

"ab"
efgh
```

## bufio.Scanner

ReadBytes('\n') or ReadString('\n') or ReadLine or Scanner?

### 1. ReadBytes 不会自动处理换行, 将换行符移除

```
s := strings.NewReader("a\r\nb")
r := bufio.NewReader(s)
for {
    token, _, err := r.ReadLine()
    if len(token) > 0 {
        fmt.Printf("Token (ReadLine): %q\n", token)
    }
    if err != nil {
        break
    }
}
s.Seek(0, io.SeekStart)
r.Reset(s)
for {
    token, err := r.ReadBytes('\n')
    fmt.Printf("Token (ReadBytes): %q\n", token)
    if err != nil {
        break
    }
}
s.Seek(0, io.SeekStart)
scanner := bufio.NewScanner(s)
for scanner.Scan() {
    fmt.Printf("Token (Scanner): %q\n", scanner.Text())
}
Token (ReadLine): "a"
Token (ReadLine): "b"
Token (ReadBytes): "a\r\n"
Token (ReadBytes): "b"
Token (Scanner): "a"
Token (Scanner): "b"
```

### 2. ReadLine 不能处理一行比 buf 大的情况

```
s := strings.NewReader(strings.Repeat("a", 20) + "\n")
r := bufio.NewReaderSize(s, 16)
token, _, _ := r.ReadLine()
fmt.Printf("Token (ReadLine): \t%q\n", token)
s.Seek(0, io.SeekStart)
r.Reset(s)
token, _ = r.ReadBytes('\n')
fmt.Printf("Token (ReadBytes): \t%q\n", token)
s.Seek(0, io.SeekStart)
scanner := bufio.NewScanner(s)
scanner.Scan()
fmt.Printf("Token (Scanner): \t%q\n", scanner.Text())

Token (ReadLine): 	"aaaaaaaaaaaaaaaa"
Token (ReadBytes): 	"aaaaaaaaaaaaaaaaaaaa\n"
Token (Scanner): 	"aaaaaaaaaaaaaaaaaaaa"
```

ReadLine 需要多次 I/O，才能读完 Read stream，而 Scanner 默认是 [64 * 1024](https://github.com/golang/go/blob/436f2d8d974954ef052f1b71c751df713704ab00/src/bufio/scan.go#L74)。如果一行超过 Scanner 限制，将不会处理，直接报错 bufio.Scanner: token too long

如果在处理超长文本，读取一行的时候：

- Scanner 会自动处理一行很长的情况
- ReadLine 需要程序员调用多次，手动管理
- ReadBytes 没有限制，但是用起来贼麻烦

```
s := strings.NewReader(strings.Repeat("a", 64*1024) + "\n")
r := bufio.NewReader(s)
token, _, err := r.ReadLine()
fmt.Printf("Token (ReadLine): %d\n", len(token))
fmt.Printf("Error (ReadLine): %v\n", err)
s.Seek(0, io.SeekStart)
r.Reset(s)
token, err = r.ReadBytes('\n')
fmt.Printf("Token (ReadBytes): %d\n", len(token))
fmt.Printf("Error (ReadBytes): %v\n", err)
s.Seek(0, io.SeekStart)
scanner := bufio.NewScanner(s)
scanner.Scan()
fmt.Printf("Token (Scanner): %d\n", len(scanner.Text()))
fmt.Printf("Error (Scanner): %v\n", scanner.Err())

Token (ReadLine): 4096
Error (ReadLine): <nil>
Token (ReadBytes): 65537
Error (ReadBytes): <nil>
Token (Scanner): 0
Error (Scanner): bufio.Scanner: token too long
```

### 3. Scanner 的 API 更加友好，更好用
[API](https://golang.org/pkg/bufio/#Scanner)

```
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		fmt.Println(scanner.Text()) // Println will add back the final '\n'
	}
	if err := scanner.Err(); err != nil {
		fmt.Fprintln(os.Stderr, "reading standard input:", err)
	}
}
```

## bufio 目前在那里用？

官方 package 大量使用 bufio 进行 I/O 优化，可以参考

- archive/zip
- compress/*
- encoding/*
- image/*
- net/http
- sync.Pool

net/http 中 TCP connections 读取，就是使用 bufio 进行 [优化](https://github.com/golang/go/blob/1e3f563b145ad98d2a5fcd4809e25a6a0bc8f892/src/net/http/transport.go#L1193)

sync.Pool 为了减少 GC，也是使用 bufio 进行 [优化](https://github.com/golang/go/blob/1e3f563b145ad98d2a5fcd4809e25a6a0bc8f892/src/net/http/server.go#L796)
