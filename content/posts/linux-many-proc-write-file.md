---
title: Linux 中多个进程操作同一个文件时会发生什么
date: 2019-03-27 14:24:23
tags: [Go, Linux]
---

[与 Windows 不同](https://softwareengineering.stackexchange.com/questions/288025/reading-file-during-write-on-linux)， Linux 允许一个文件在写入的时候被读取（或者在被读取的时候写入），本文就来探索一下多个进程同时读写同一个文件会产生的效果。

<!--more-->

## Read + Read

多个进程同时读取同一个文件不会出现问题的，放心去干吧。

## Read + Write

本文的重点研究对象。Linux 通过文件描述符表维护了打开的文件描述符信息，而文件描述符表中的每一项都指向一个内核维护的文件表，文件表指向打开的文件的 [vnode](https://www.freebsd.org/cgi/man.cgi?query=vnode)(Unix) 和 [inode](https://en.wikipedia.org/wiki/Inode)。同时，文件表保存了进程对文件读写的偏移量等信息。

我们通过两个简单的 Go 语言程序来测试一下在读文件的同时修改文件会发生什么：

**testwrite.go**

```go
func writeFile(filename string, data string) {
	f, _ := os.OpenFile(filename, os.O_CREATE|os.O_WRONLY, 0644)
	defer f.Close()
	body := []byte(data)
	_, _ = f.Write(body)
}

func main() {
	// 首先向文件中写入 “Hello World!”
	writeFile("test.txt", "Hello World!")
	time.Sleep(7 * time.Second)
	// 七秒后，修改文件内容，写入 “Author MeiK!”
	writeFile("test.txt", "Author MeiK!")
}
```

**testread.go**

```go
func readFile(filename string) {
	f, _ := os.OpenFile(filename, unix.O_RDONLY, 0644)
	defer f.Close()

	body := make([]byte, 1)
	n := 1
	for n != 0 {
		time.Sleep(time.Second)
		var err error
		n, err = f.Read(body)
		if err == io.EOF {
			break
		}
		s, _ := f.Seek(0, os.SEEK_CUR)
		fmt.Printf("%c %d\n", body, s)
	}
}

func main() {
	readFile("test.txt")
}
```

同时执行两个程序：

```shell
./testwrite & ./testread
```

输出：

```
[H] 1
[e] 2
[l] 3
[l] 4
[o] 5
[ ] 6
[ ] 7
[M] 8
[e] 9
[i] 10
[K] 11
[!] 12
```

这个程序打印了读取到的内容以及读取到每一步的文件偏移量。我们首先写入 `Hello World!`，开始每秒读取一个字符，并且在 7 秒后重新将 `Author MeiK!` 写入文件。我们最终读取到了什么呢？既不是 `Hello World!`，也不是 `Author MeiK!`，而是 `Hello  MeiK!`。我们每个字符串读取到了一半！

从每一步的文件偏移量来看，读取的程序只是按部就班的一个字符一个字符的读取文件，对文件内容的变化毫无感知，当读取到文件结尾的 `EOF` 时结束读取。

那么我们要如何保证读取与写入的一致性呢？ Linux 提供了 `fcntl` 系统调用，可以[锁定文件](https://www.gnu.org/software/libc/manual/html_node/File-Locks.html)。

我们对刚刚的文件稍作修改，使用 `fcntl` 进行加锁：

**testwrite.go**

```go
func writeFile(filename string, data string) {
	fmt.Println("write start")
	f, _ := os.OpenFile(filename, os.O_CREATE|os.O_WRONLY, 0644)
	defer f.Close()

	flockT := unix.Flock_t{
		Type:   unix.F_WRLCK,
		Whence: io.SeekStart,
		Start:  0,
		Len:    0,
	}
	_ = unix.FcntlFlock(f.Fd(), unix.F_SETLKW, &flockT)

	body := []byte(data)
	_, _ = f.Write(body)
	fmt.Println("write end")
}

func main() {
	// 首先向文件中写入 “Hello World!”
	writeFile("test.txt", "Hello World!")
	time.Sleep(7 * time.Second)
	// 七秒后，修改文件内容，写入 “Author MeiK!”
	writeFile("test.txt", "Author MeiK!")
}
```

**testread.go**

```go
func readFile(filename string) {
	f, _ := os.OpenFile(filename, unix.O_RDONLY, 0644)
	defer f.Close()

	flockT := unix.Flock_t{
		Type:   unix.F_RDLCK,
		Whence: io.SeekStart,
		Start:  0,
		Len:    0,
	}
	_ = unix.FcntlFlock(f.Fd(), unix.F_SETLKW, &flockT)

	body := make([]byte, 1)
	n := 1
	for n != 0 {
		time.Sleep(time.Second)
		var err error
		n, err = f.Read(body)
		if err == io.EOF {
			break
		}
		s, _ := f.Seek(0, os.SEEK_CUR)
		fmt.Printf("%c %d\n", body, s)
	}
}

func main() {
	readFile("test.txt")
}
```

额外添加 `write start` 和 `write end` 来标识当前进度，执行结果如下：

```
write start
write end
[H] 1
[e] 2
[l] 3
[l] 4
[o] 5
[ ] 6
write start
[W] 7
[o] 8
[r] 9
[l] 10
[d] 11
[!] 12
write end
```

可以看到，第一次写入文件时，进程很快的完成了写入；而当第二次写入时，由于此时 `read` 进程对文件加锁了，导致写入进程阻塞，直到读取结束后， `write` 进程才把内容写入了文件。因此 `read` 进程读取到的就是第一次写入的内容 `Hello World!`。完美的解决了我们的问题，可喜可贺。

不过，还有两点需要注意：

1. 文件锁是与进程相关的，一个进程中的多个线程/协程对同一个文件进行的锁操作会互相覆盖掉，从而无效。
2. `fcntl` 创建的锁是建议性锁，只有写入的进程和读取的进程都遵循建议才有效；对应的有强制性锁，会在每次文件操作时进行判断，但性能较差，因此 Linux/Unix 系统默认采用的是建议性锁。

关于不同类型锁之间的交互，可以参照此表：

| 当前状态 | 加读锁 | 加写锁 |
| --- | --- | --- |
| 无锁 | 允许 | 允许 |
| 一个或多个读锁 | 允许 | 拒绝 |
| 一个写锁 | 拒绝 | 拒绝 |

## Write + Write

两个进程同时写入可以和 `Write + Read` 一样靠加锁来解决同步问题，不过还有其他的解决方案：假如我们现在有多个进程在将日志写入同一个日志文件，那么我们可以使用 `O_APPEND` 标志来打开文件，这样在每次写入时都会 `lseek` 到文件末尾进行写入，这是一个原子操作，因此不会产生同步问题。

## 结论

如果一个文件在读取的同时被修改（而没有添加任何锁机制的话），那么将可能会读到错误的数据，很多时候这比读不到数据或读到旧版本的数据的后果更加可怕……因此，如果有准确性要求较高的文件读取的情景的话，最好还是用强制性锁对文件进行保护。
