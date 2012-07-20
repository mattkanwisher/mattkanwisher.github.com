---
layout: post
title: "Handling blocking IO in GO."
date: 2012-07-19 20:17
comments: true
categories: 
---


So I was trying to write a simple system monitoring server in (Go)[http://golang.org/] and one of the problems I ran into was blocking IO, so for example if you need to spawn a sub process and listen to it's output you need to spawn a seperate go routines to monitor it. Thats fine, however the problem is if you ever have a piece of blocking IO that never returns like say your spawning "tail -f" you will have no way of killing the goroutine. I love blocking IO, but not having a mechanism to timeout on blocking IO seems like a big deficiency. So lets start with a really simple example of how the blocking IO works and build up to solve my problem.


``` go
  cmd := exec.Command("tail", "-f", "/tmp/matt.txt")
  stdout, err := cmd.StdoutPipe()
  if err != nil {
      log.Fatal(err)
  }
  if err := cmd.Start(); err != nil {
      log.Fatal(err)
  }


  //Ok here is the read
  reader := bufio.NewReader(stdout)
  s,err := reader.ReadString('\n')

```


Ok thats great, but now the whole process waits while you read the io, which sucks if you have a server with lots of clients connecting. 
So lets improve it a bit and make it a go routine so it doesn't block at all.

``` go

func readLogData(filename string, log_id int, logOutputChan chan<- *LogTuple, deathChan chan<- *string) {
    cmd := exec.Command("tail", "-f", filename)
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        log.Fatal(err)
    }
    if err := cmd.Start(); err != nil {
        log.Fatal(err)
    }

    err  = nil
    reader := bufio.NewReader(stdout)
    sbuffer := ""
    lines := 0
    for ; err == nil;  {
        s,err := reader.ReadString('\n')
        sbuffer += s
        lines += 1
        if(lines > 5 ) { //|| time > 1 min) {
            logOutputChan <- &LogTuple{ log_id, sbuffer}
            sbuffer = ""
            lines = 0
        }
        if err != nil {
            deathChan <- &filename
            return
        }
    }

    //SetFinalizer
}

go readLogData(alog.Log.Path, alog.Log.Id, logOutputChan, gorDeathChan)
```

A little bit more complicated, but lets explain it, so we broke this code into a seperate function that takes in 2 channels. Channels in go let you send data between go routines. We have one channel to get data from the go routine, and another that tells the goroutine to die. Why we need this? Cause there is no way to kill a goroutine safely.

First problem you might notice is that if the underlying IO blocks forever and never returns we can never end this go routine. When would this happen? A good example is "tail -f " or a process that has gone haywire and is not responding. So how do we fix this ?


``` go
package main

import (
	"fmt"
	"log"
	"os/exec"
	"time"
)

func main() {
	cmd := exec.Command("tail", "-f", "/tmp/foo.txt")
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		log.Fatal(err)
	}
	if err := cmd.Start(); err != nil {
		log.Fatal(err)
	}

	ch := make(chan string)
	quit := make(chan bool)
	go func() {
		buf := make([]byte, 1024)
		for {
			n, err := stdout.Read(buf)
			if n != 0 {
				ch <- string(buf[:n])
			}
			if err != nil {
				break
			}
		}
		fmt.Println("Goroutine finished")
		close(ch)
	}()

	time.AfterFunc(time.Second, func() { quit <- true })

loop:
	for {
		select {
		case s, ok := <-ch:
			if !ok {
				break loop
			}
			fmt.Print(s)
		case <-quit:
			cmd.Process.Kill()
		}
	}
}
```

So basically we kill the underlying process so the IO sends an EOF. This is a simplified example. In my server I do keep track of when I want processes to die and spawn and its not on a simple timer.

Special thanks to mrlauer on the [Golang-nuts](https://groups.google.com/forum/?fromgroups#!forum/golang-nuts) message board.

Shameless Plug:

At [Errplane](http://errplane.com) we use Go to build a service for rails exception tracking, log aggregation and alerting / monitoring.

