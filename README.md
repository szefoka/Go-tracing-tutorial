# Go tracing

## Install Go

Before everything you shall install Go if you haven't done it yet
```
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt-get update
sudo apt-get install golang-go
```

# go tool trace
To make it up and running:
Download the go repository from github
```
git clone https://github.com/golang/go.git
```
step into the newly downloaded directory, and copy the misc directory to $(go env GOROOT)/
```
cp misc $(go env GOROOT)/
```
Now you can see the order of goroutines and events running in your go program

To see the programs flowgraph and times consumed, you shall install the graphviz and dot libraries
```
sudo apt install python-pydot python-pydot-ng graphviz 
```
This is for debian based computers, so maybe you need to issue the corresponding commands on other kind of distros

To be able to record your program's traces you shall insert some tracing related lines to your code, for this here is an example

```
package main

import (
	"os"
	"runtime/trace"
)

func main() {
	f, err := os.Create("trace.out")
	if err != nil {
		panic(err)
	}
	defer f.Close()

	err = trace.Start(f)
	if err != nil {
		panic(err)
	}
	defer trace.Stop()

  // Your program here
}
```
This small program generates the trace.out file whic contains the program's trace, to visalize it issue the following:
```
go tool trace -http <IP:PORT> trace.out
```

## pprof

pprof is for generating flamegraphs and for deeper examining of your code

First you need to install pprof
```
go get github.com/google/pprof
```

To use pprof for generating more verbose output of your traces you shall copy the url of from the web UI of the go trace tool related to the questionable program trace and you can start examining it.
```
pprof -http "0.0.0.0:8081" 'http://hp045.utah.cloudlab.us:8080/io?id=6436192&raw=1'
```

## go tool pprof

go tool pprof is for profiling your application, you can check the CPU or memory consumtion of each function in the call tree. It also gives the facilty to visualize our traces on a fancy web UI
Before anything, you gotta extend your application to let go tool pprof accessing it.

I've just stolen this code from Julia Evans' blog (she uses this to trace memory leaks)

```
package main

import (
        "fmt"
        "log"
        "net/http"
        _ "net/http/pprof"
        "time"
        "sync"
        // "github.com/alexellis/golang-http-template/template/golang-http/function"
)

func main() {
    // we need a webserver to get the pprof webserver
    go func() {
        log.Println(http.ListenAndServe("0.0.0.0:6060", nil))
    }()
    fmt.Println("hello world")
    var wg sync.WaitGroup
    wg.Add(1)
    go leakyFunction(wg)
    wg.Wait()
}

func leakyFunction(wg sync.WaitGroup) {
    defer wg.Done()
    s := make([]string, 3)
    for i:= 0; i < 10000000; i++{
        s = append(s, "magical pandas")
        if (i % 100000) == 0 {
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

After extending your application's code, you can start recording the running of your application by issuing the command below, which records the trace for 5 seconds
```
go tool pprof localhost:6060/debug/pprof/profile?seconds=5
```
From the output you can locate the the recorded profile of your application.
```
Fetching profile over HTTP from http://localhost:6060/debug/pprof/profile?seconds=5
Saved profile in /root/pprof/pprof.gows.samples.cpu.002.pb.gz
File: gows
Type: cpu
Time: May 27, 2019 at 8:02am (MDT)
Duration: 5.38s, Total samples = 12.06s (224.03%)
Entering interactive mode (type "help" for commands, "o" for options)
```
Pressing Ctrl+D exits you from the command prompt
To use webui of go tool pprof you need to issue the following command

```
go tool pprof -http 0.0.0.0:8080 /root/pprof/pprof.gows.samples.cpu.002.pb.gz
```

## References:
https://github.com/golang/go/wiki/Ubuntu

https://medium.com/@cep21/using-go-1-10-new-trace-features-to-debug-an-integration-test-1dc39e4e812d

https://making.pusher.com/go-tool-trace/

https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/
