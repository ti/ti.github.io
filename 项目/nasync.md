---
catalog: projects
tags: [golang]
github: https://github.com/ti/nasync
link: https://github.com/ti/nasync
date: 2016-02-04
picture: img/number.jpg
---
# NASYNC 可自定义容量的异步库

a customizable async task pool for golang, (event bus, runtime)

github: https://github.com/ti/nasync

## Fetures

* less memory
* more effective
* max gorutines and memory customizable
* more safe


## Simple Usage

```go
nasync.Do(function)
```


## Advanced Usage

```bash
go get github.com/ti/nasync
```
```go
import "github.com/ti/nasync"

func main() {
        //new a async pool in max 1000 task in max 1000 gorutines
        async := nasync.New(1000,1000)
        defer async.Close()
        async.Do(doSometing,"hello word")
        nasync.Do(func() {
			http.Get("https://github.com/ti/")
		})
}


func doSometing(msg string) string{
	return "i am done by " + msg
}


```

# WHY

golang is something easy but fallible language, you may do this 

```go
   func yourfucntion() {
            go dosomething()  // this will got error on high load
    }
```

you may get "too many open files" error, when your application  in High load, so you need this, you can do any thing in async by this, it is trusty。your can use this for:

* http or file writer logging
* improve main thread speed
* limited background task pool

## What if something callback ?

```go
import "github.com/ti/nasync"

func main() {
        nasync.Do(func() {
        		result := doSometing("msg")
        		fmt.Println("i am call back by ",result)
        })
}

func doSometing(msg string) string{
	return "i am done by " + msg
}

```

