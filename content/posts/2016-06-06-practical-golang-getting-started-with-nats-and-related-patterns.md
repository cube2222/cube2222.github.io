---
title: 'Practical Golang: Getting started with NATS and related patterns'
author: Jacob Martin
type: legacy-posts
date: 2016-06-06T11:51:48+00:00
url: /2016/06/06/practical-golang-getting-started-with-nats-and-related-patterns/
categories:
  - Go
  - Practical Golang
tags:
  - event
  - go
  - golang
  - master
  - message
  - microservice
  - NATS
  - pattern
  - protobuf
  - pub
  - slave
  - sub
  - subscribe
  - worker

---
# Practical Golang: Getting started with NATS and related patterns

## Introduction

Microservices&#8230; the never disappearing buzzword of our times. They promise a lot, but can be slow or complicated if not implemented correctly. One of the main challenges when developing and using a microservice-based architecture is getting the _communication_ right. Many will ask, why not **REST**? As I did at some point. Many will actually use it. But the truth is that it leads to tighter coupling, and is _synchronous_. Microservice architectures are meant to be asynchronous. Also, REST is blocking, which also isn&#8217;t good on many occasions.

What are we meant to use for communication? Usually we use:  
&#8211; RPC &#8211; Remote Procedure Call  
&#8211; Message BUS/Broker

In this article I&#8217;ll write about one specific Message BUS called **NATS** and using it in Go.

There are also other message BUS&#8217;ses/Brokers. Some popular ones are **Kafka** and **RabbitMQ**.

Why NATS? It&#8217;s simple, and astonishingly fast.

## Setting up NATS

To use NATS you can do one of the following things:  
1. Use the [NATS Docker image][1]  
2. Get the [binaries][2]  
3. Use the public NATS server _nats://demo.nats.io:4222_  
4. Build from [source][3]

Also, remember to

    go get https://github.com/nats-io/nats
    

the official Go library.

## Getting started

In this article we&#8217;ll be using protobuffs a lot. So if you want to know more about them, check out my [previous article about protobuffs][4].

First, let&#8217;s write one of the key usages of microservices. A fronted, that lists information from other micrservices, but doesn&#8217;t care if one of them is down. It will respond to the user anyways. This makes microservices swappable live, one at a time.

In each of our services we&#8217;ll need to connect to NATS:

```go
package main

import (
    "github.com/nats-io/nats"
    "fmt"
)

var nc *nats.Conn

func main() {

    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }
    var err error

    nc, err = nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println(err)
    }
}
```

Now, let&#8217;s write the first provider service. It will receive a _User Id_, and answer with a _user name_ For which we&#8217;ll need a transport structure to send its data over _NATS_. I wrote this short proto file for that:

```proto
syntax = "proto3";
package Transport;

message User {
        string id = 1;
        string name = 2;
}
```

Now we will create the map containing our user names:

```go
var users map[string]string
var nc *nats.Conn

func main() {

    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }
    var err error

    nc, err = nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println(err)
    }

    users = make(map[string]string)
    users["1"] = "Bob"
    users["2"] = "John"
    users["3"] = "Dan"
    users["4"] = "Kate"
}
```

and finally the part that&#8217;s most interesting to us. Subscribing to the topic:

```go
users["4"] = "Kate"

nc.QueueSubscribe("UserNameById", "userNameByIdProviders", replyWithUserId)
```

Notice that it&#8217;s a **QueueSubscribe**. Which means that if we start 10 instances of this service in the _userNameByIdProviders_ group , only one will get each message sent over _UserNameById_. Another thing to note is that this function call is asynchronous, so we need to block somehow. This _select {}_ will provide an endless block:

```go
nc.QueueSubscribe("UserNameById", "userNameByIdProviders", replyWithUserId)
select {}
}
```

Ok, now to the _replyWithUserId_ function:

```go
func replyWithUserId(m *nats.Msg) {
}
```

Notice that it takes one argument, a pointer to the message.

We&#8217;ll unmarshal the data:

```go
func replyWithUserId(m *nats.Msg) {

    myUser := Transport.User{}
    err := proto.Unmarshal(m.Data, &myUser)
    if err != nil {
        fmt.Println(err)
        return
}
```

get the name and marshal back:

```go
myUser.Name = users[myUser.Id]
data, err := proto.Marshal(&myUser)
if err != nil {
    fmt.Println(err)
    return
}
```

And, as this shall be a request we&#8217;re handling, we respond to the _Reply topic_, a topic created by the caller exactly for this purpose:

```go
if err != nil {
    fmt.Println(err)
    return
}
fmt.Println("Replying to ", m.Reply)
nc.Publish(m.Reply, data)

}
```

Ok, now let&#8217;s get to the second service. Our time provider service, first the same basic structure:

```go
package main

import (
    "github.com/nats-io/nats"
    "fmt"
    "github.com/cube2222/Blog/NATS/FrontendBackend"
    "github.com/golang/protobuf/proto"
    "os"
    "sync"
    "time"
)

// We use globals because it's a small application demonstrating NATS.

var nc *nats.Conn

func main() {

    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }
    var err error

    nc, err = nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println(err)
    }

    nc.QueueSubscribe("TimeTeller", "TimeTellers", replyWithTime)
    select {} // Block forever
}
```

This time we&#8217;re not getting any data from the caller, so we just marshal our time into this proto structure:

```proto
syntax = "proto3";
package Transport;

message Time {
        string time = 1;
}
```

and send it back:

```go
func replyWithTime(m *nats.Msg) {
    curTime := Transport.Time{time.Now().Format(time.RFC3339)}

    data, err := proto.Marshal(&curTime)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("Replying to ", m.Reply)
    nc.Publish(m.Reply, data)

}
```

We can now get to our frontend, which will use both those services. First the standard basic structure:

```go
package main

import (
    "net/http"
    "github.com/gorilla/mux"
    "github.com/cube2222/Blog/NATS/FrontendBackend"
    "github.com/golang/protobuf/proto"
    "fmt"
    "github.com/nats-io/nats"
    "time"
    "os"
    "sync"
)

var nc *nats.Conn

func main() {
    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }
    var err error

    nc, err = nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println(err)
    }

    m := mux.NewRouter()
    m.HandleFunc("/{id}", handleUserWithTime)

    http.ListenAndServe(":3000", m)
}
```

That&#8217;s a pretty standard web server, now to the interesting bits, the _handleUserWithTime_ function, which will respond with the user name and time:

```go
func handleUserWithTime(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    myUser := Transport.User{Id: vars["id"]}
    curTime := Transport.Time{}
    wg := sync.WaitGroup{}
    wg.Add(2)
}
```

We&#8217;ve parsed the request arguments and started a _WaitGroup_ with the value two, as we will do one _asynchronous_ request for each of our services. First we&#8217;ll marshal the user struct:

```go
go func() {
    data, err := proto.Marshal(&myUser)
    if err != nil || len(myUser.Id) == 0 {
        fmt.Println(err)
        w.WriteHeader(500)
        fmt.Println("Problem with parsing the user Id.")
        return
    }
```

and, then we make a request. Sending the user data, and waiting at most 100 ms for the response:

```go
fmt.Println("Problem with parsing the user Id.")
return
}

msg, err := nc.Request("UserNameById", data, 100 * time.Millisecond)
```

now we can check if any error happend, or the response is empty and finish this thread:

```go
msg, err := nc.Request("UserNameById", data, 100 * time.Millisecond)
if err == nil && msg != nil {
    myUserWithName := Transport.User{}
    err := proto.Unmarshal(msg.Data, &myUserWithName)
    if err == nil {
        myUser = myUserWithName
    }
}
wg.Done()
}()
```

Next we&#8217;ll do the request to the _Time Tellers_.  
We again make a request, but its body is nil, as we don&#8217;t need to pass any data:

```go
go func() {
    msg, err := nc.Request("TimeTeller", nil, 100*time.Millisecond)
    if err == nil && msg != nil {
        receivedTime := Transport.Time{}
        err := proto.Unmarshal(msg.Data, &receivedTime)
        if err == nil {
            curTime = receivedTime
        }
    }
    wg.Done()
}()
```

After both requests finished (or failed) we can just respond to the user:

```go
wg.Wait()

fmt.Fprintln(w, "Hello ", myUser.Name, " with id ", myUser.Id, ", the time is ", curTime.Time, ".")
}
```

Now if you actually test it, you&#8217;ll notice that if one of the provider services isn&#8217;t active, the frontend will respond anyways, putting a zero&#8217;ed value in place of the non-available resource. You could also make a template that shows an error in that place.

Ok, that was already an interesting architecture. Now we can implement&#8230;

## The Master-Slave pattern

This is such a popular pattern, especially in Go, that we really should know how to implement it. The workers will do simple operations on a text file (count the usage amounts of each word in a comma-separated list).

Now you could think that the _Master_, should send the files to the _Workers_ over NATS. Wrong. This would lead to a huge slowdown of NATS (at least for bigger files). That&#8217;s why the _Master_ will send the files to a file server over a REST API, and the _Workers_ will get it from there. We&#8217;ll also learn how to do _service discovery_ over NATS.

First, the _File Server_. I won&#8217;t really go through the file handling part, as it&#8217;s a simple get/post API.I will however, go over the _service discovery_ part.

```go
package main

import (
    "net/http"
    "github.com/gorilla/mux"
    "os"
    "io"
    "fmt"
    "github.com/nats-io/nats"
    "github.com/cube2222/Blog/NATS/MasterWorker"
    "github.com/golang/protobuf/proto"
)

func main() {

    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }

    m := mux.NewRouter()

    m.HandleFunc("/{name}", func(w http.ResponseWriter, r *http.Request) {
        vars := mux.Vars(r)
        file, err := os.Open("/tmp/" + vars["name"])
        defer file.Close()
        if err != nil {
            w.WriteHeader(404)
        }
        if file != nil {
            _, err := io.Copy(w, file)
            if err != nil {
                w.WriteHeader(500)
            }
        }
    }).Methods("GET")

    m.HandleFunc("/{name}", func(w http.ResponseWriter, r *http.Request) {
        vars := mux.Vars(r)
        file, err := os.Create("/tmp/" + vars["name"])
        defer file.Close()
        if err != nil {
            w.WriteHeader(500)
        }
        if file != nil {
            _, err := io.Copy(file, r.Body)
            if err != nil {
                w.WriteHeader(500)
            }
        }
    }).Methods("POST")

    RunServiceDiscoverable()

    http.ListenAndServe(":3000", m)
}
```

Now, what does the _RunServiceDiscoverable_ function do? It connects to the NATS server and responds with its own http address to incoming requests.

```go
func RunServiceDiscoverable() {
    nc, err := nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println("Can't connect to NATS. Service is not discoverable.")
    }
    nc.Subscribe("Discovery.FileServer", func(m *nats.Msg) {
        serviceAddressTransport := Transport.DiscoverableServiceTransport{"http://localhost:3000"}
        data, err := proto.Marshal(&serviceAddressTransport)
        if err == nil {
            nc.Publish(m.Reply, data)
        }
    })
}
```

The proto file looks like this:

```proto
syntax = "proto3";
package Transport;

message DiscoverableServiceTransport {
        string Address = 1;
}
```

We can now go on with the _Master_.

The protofile for the _Task_ structure is:

```proto
syntax = "proto3";
package Transport;

message Task {
        string uuid = 1;
        string finisheduuid = 2;
        int32 state = 3; // 0 - not started, 1 - in progress, 2 - finished
        int32 id = 4;
}
```

Our _Master_ will hold a list of tasks with the respecting **UUID** (at the same time the name of the file), id (the position in the master Tasks slice), and a pointer which holds the position of the last not finished Task, which will get updated on new Task retrieval. It&#8217;s pretty similar to the Task storage in my [Microservice Architecture series][5]

I&#8217;m using _github.com/satori/go.uuid_ for **UUID** generation.

First, as usual, the basic structure:

```go
package main

import (
    "github.com/satori/go.uuid"
    "github.com/cube2222/Blog/NATS/MasterWorker"
    "os"
    "fmt"
    "github.com/nats-io/nats"
    "github.com/golang/protobuf/proto"
    "time"
    "bytes"
    "net/http"
    "sync"
)

var Tasks []Transport.Task
var TaskMutex sync.Mutex
var oldestFinishedTaskPointer int
var nc *nats.Conn


func main() {
    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }

    var err error

    nc, err = nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println(err)
    }

    Tasks = make([]Transport.Task, 0, 20)
    TaskMutex = sync.Mutex{}
    oldestFinishedTaskPointer = 0

    initTestTasks()

    wg := sync.WaitGroup{}

    nc.Subscribe("Work.TaskToDo", func (m *nats.Msg) {
    })

    nc.Subscribe("Work.TaskFinished", func (m *nats.Msg) {
    })

    select {} // Block forever
}
```

Ok, we&#8217;ve also already set up the _Subscriptions_

How does the _initTestTasks_ function work? It&#8217;s interesting because it gets the file server address over NATS.

So, we want to create 20 test Tasks, so we run the loop 20 times:

```go
func initTestTasks() {
    for i := 0; i &lt; 20; i++ {
    }
}
```

We create a new _Task_ and ask the _File Server_ for its address:

```go
for i := 0; i &lt; 20; i++ {
    newTask := Transport.Task{Uuid: uuid.NewV4().String(), State: 0}
    fileServerAddressTransport := Transport.DiscoverableServiceTransport{}
    msg, err := nc.Request("Discovery.FileServer", nil, 1000 * time.Millisecond)
    if err == nil && msg != nil {
        err := proto.Unmarshal(msg.Data, &fileServerAddressTransport)
        if err != nil {
            continue
        }
    }
    if err != nil {
        continue
    }

    fileServerAddress := fileServerAddressTransport.Address
}
```

Next we finally make the post Request to the file server and add the Task to our Tasks list:

```go
        fileServerAddress := fileServerAddressTransport.Address
        data := make([]byte, 0, 1024)
        buf := bytes.NewBuffer(data)
        fmt.Fprint(buf, "get,my,data,my,get,get,have")
        r, err := http.Post(fileServerAddress + "/" + newTask.Uuid, "", buf)
        if err != nil || r.StatusCode != http.StatusOK {
            continue
        }

        newTask.Id = int32(len(Tasks))
        Tasks = append(Tasks, newTask)
    }
```

How do we dispatch new Tasks to do? Simply like this:

```go
nc.Subscribe("Work.TaskToDo", func (m *nats.Msg) {
    myTaskPointer, ok := getNextTask()
    if ok {
        data, err := proto.Marshal(myTaskPointer)
        if err == nil {
            nc.Publish(m.Reply, data)
        }
    }
})
```

How do we get the next Task? We just loop over the Task to find one that is not started. If tasks above our pointer are all finished, then we also move up the pointer. Remember the mutex as this function may be run in parallel:

```go
func getNextTask() (*Transport.Task, bool) {
    TaskMutex.Lock()
    defer TaskMutex.Unlock()
    for i := oldestFinishedTaskPointer; i &lt; len(Tasks); i++ {
        if i == oldestFinishedTaskPointer && Tasks[i].State == 2 {
            oldestFinishedTaskPointer++
        } else {
            if Tasks[i].State == 0 {
                Tasks[i].State = 1
                go resetTaskIfNotFinished(i)
                return &Tasks[i], true
            }
        }
    }
    return nil, false
}
```

We also called the _resetTaskIfNotFinished_ function. It will reset the Task state if it&#8217;s still in progress after 2 minutes:

```go
func resetTaskIfNotFinished(i int) {
    time.Sleep(2 * time.Minute)
    TaskMutex.Lock()
    if Tasks[i].State != 2 {
        Tasks[i].State = 0
    }
}
```

The TaskFinished subscription handler is much simpler, it just sets the Task to finished, and the **UUID** accordingly to the received protobuffer:

```go
nc.Subscribe("Work.TaskFinished", func (m *nats.Msg) {
    myTask := Transport.Task{}
    err := proto.Unmarshal(m.Data, &myTask)
    if err == nil {
        TaskMutex.Lock()
        Tasks[myTask.Id].State = 2
        Tasks[myTask.Id].Finisheduuid = myTask.Finisheduuid
        TaskMutex.Unlock()
    }
})
```

That&#8217;s all in regards to the _Master_! We can now move on writing the _Worker_.

The basic structure:

```go
package main

import (
    "os"
    "fmt"
    "github.com/nats-io/nats"
    "time"
    "github.com/cube2222/Blog/NATS/MasterWorker"
    "github.com/golang/protobuf/proto"
    "net/http"
    "bytes"
    "io/ioutil"
    "sort"
    "strings"
    "github.com/satori/go.uuid"
    "sync"
)

var nc *nats.Conn

func main() {
    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }

    var err error

    nc, err = nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println(err)
    }

    for i := 0; i &lt; 8; i++ {
        go doWork()
    }

    select {} // Block forever
}
```

Now the main function doing something here is the _doWork_ function. I&#8217;ll post it all at once with comments everywhere, as it&#8217;s a very long function and this will be the most convenient way to read it:

```go
func doWork() {
    for {
        // We ask for a Task with a 1 second Timeout
        msg, err := nc.Request("Work.TaskToDo", nil, 1 * time.Second)
        if err != nil {
            fmt.Println("Something went wrong. Waiting 2 seconds before retrying:", err)
            continue
        }

        // We unmarshal the Task
        curTask := Transport.Task{}
        err = proto.Unmarshal(msg.Data, &curTask)
        if err != nil {
            fmt.Println("Something went wrong. Waiting 2 seconds before retrying:", err)
            continue
        }

        // We get the FileServer address
        msg, err = nc.Request("Discovery.FileServer", nil, 1000 * time.Millisecond)
        if err != nil {
            fmt.Println("Something went wrong. Waiting 2 seconds before retrying:", err)
            continue
        }

        fileServerAddressTransport := Transport.DiscoverableServiceTransport{}
        err = proto.Unmarshal(msg.Data, &fileServerAddressTransport)
        if err != nil {
            fmt.Println("Something went wrong. Waiting 2 seconds before retrying:", err)
            continue
        }

        // We get the file
        fileServerAddress := fileServerAddressTransport.Address
        r, err := http.Get(fileServerAddress + "/" + curTask.Uuid)
        if err != nil {
            fmt.Println("Something went wrong. Waiting 2 seconds before retrying:", err)
            continue
        }

        data, err := ioutil.ReadAll(r.Body)
        if err != nil {
            fmt.Println("Something went wrong. Waiting 2 seconds before retrying:", err)
            continue
        }

        // We split and count the words
        words := strings.Split(string(data), ",")
        sort.Strings(words)
        wordCounts := make(map[string]int)
        for i := 0; i &lt; len(words); i++{
            wordCounts[words[i]] = wordCounts[words[i]] + 1
        }

        resultData := make([]byte, 0, 1024)
        buf := bytes.NewBuffer(resultData)

        // We print the results to a buffer
        for key, value := range wordCounts {
            fmt.Fprintln(buf, key, ":", value)
        }

        // We generate a new UUID for the finished file
        curTask.Finisheduuid = uuid.NewV4().String()
        r, err = http.Post(fileServerAddress + "/" + curTask.Finisheduuid, "", buf)
        if err != nil || r.StatusCode != http.StatusOK {
            fmt.Println("Something went wrong. Waiting 2 seconds before retrying:", err, ":", r.StatusCode)
            continue
        }

        // We marshal the current Task into a protobuffer
        data, err = proto.Marshal(&curTask)
        if err != nil {
            fmt.Println("Something went wrong. Waiting 2 seconds before retrying:", err)
            continue
        }

        // We notify the Master about finishing the Task
        nc.Publish("Work.TaskFinished", data)
    }
}
```

Awesome, our _Master-Slave_ setup is ready, you can test it if you&#8217;d like. After you do, we can now check out the last architecture.

## The Events pattern

Imagine you have servers which keep connections to clients over websockets. You want these clients to get live news updates. With this pattern you can. We&#8217;ll also learn about a few convenient NATS client abstractions. Like using a encoded connection, or using channels for sending/receiving.

The basic architecture as usual:

```go
package main

import (
    "os"
    "fmt"
    "github.com/nats-io/nats"
    natsp "github.com/nats-io/nats/encoders/protobuf"
    "github.com/cube2222/Blog/NATS/EventSubs"
    "time"
)

func main() {
    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }

    nc, err := nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println(err)
    }
    ec, err := nats.NewEncodedConn(nc, natsp.PROTOBUF_ENCODER)
    defer ec.Close()
}
```

Wait&#8230; What&#8217;s that at the end!? It&#8217;s an encoded connection! It will automatically encode our structs into raw data. We&#8217;ll use the protobuf one, but there are a default one, a gob one and a json one too.

Here&#8217;s the protofile we&#8217;ll use:

```proto
syntax = "proto3";
package Transport;

message TextMessage {
        int32 id = 1;
        string body = 2;
}
```

Ok, how can we just publish simple event-structs? Totally intuitive, like that:

```go
defer ec.Close()

for i := 0; i &lt; 5; i++ {
    myMessage := Transport.TextMessage{Id: int32(i), Body: "Hello over standard!"}

    err := ec.Publish("Messaging.Text.Standard", &myMessage)
    if err != nil {
        fmt.Println(err)
    }
}
```

It&#8217;s a little bit counter intuitive with Requests. As the signature differs, it follows like this:

```go
err := ec.Request(topic, *body, *response, timeout)
```

So our request sending part will look like this:

```go
for i := 5; i &lt; 10; i++ {
    myMessage := Transport.TextMessage{Id: int32(i), Body: "Hello, please respond!"}

    res := Transport.TextMessage{}
    err := ec.Request("Messaging.Text.Respond", &myMessage, &res, 200 * time.Millisecond)
    if err != nil {
        fmt.Println(err)
    }

    fmt.Println(res.Body, " with id ", res.Id)

}
```

The last thing we can do is sending them via Channels, which is relatively the simplest:

```go
sendChannel := make(chan *Transport.TextMessage)
ec.BindSendChan("Messaging.Text.Channel", sendChannel)
for i := 10; i &lt; 15; i++ {
    myMessage := Transport.TextMessage{Id: int32(i), Body: "Hello over channel!"}

    sendChannel &lt;- &myMessage
}
```

Now we can write the receiving end. First the same structure as before:

```go
package main

import (
    "github.com/nats-io/nats"
    natsp "github.com/nats-io/nats/encoders/protobuf"
    "os"
    "fmt"
    "github.com/cube2222/Blog/NATS/EventSubs"
)

func main() {
    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments. Need NATS server address.")
        return
    }

    nc, err := nats.Connect(os.Args[1])
    if err != nil {
        fmt.Println(err)
    }
    ec, err := nats.NewEncodedConn(nc, natsp.PROTOBUF_ENCODER)
    defer ec.Close()
}
```

Ok, first the standard receive which is totally natural:

```go
defer ec.Close()

ec.Subscribe("Messaging.Text.Standard", func(m *Transport.TextMessage) {
    fmt.Println("Got standard message: \"", m.Body, "\" with the Id ", m.Id, ".")
})
```

Now, the responding, which has a little bit changed syntax again. As the handler function is:

```go
func (subject, reply string, m *Transport.TextMessage)
```

So the responding looks like this:

```go
ec.Subscribe("Messaging.Text.Respond", func(subject, reply string, m *Transport.TextMessage) {
    fmt.Println("Got ask for response message: \"", m.Body, "\" with the Id ", m.Id, ".")

    newMessage := Transport.TextMessage{Id: m.Id, Body: "Responding!"}
    ec.Publish(reply, &newMessage)
})
```

And finally using channels, which doesn&#8217;t differ nearly at all in comparison to the sending side:

```go
receiveChannel := make(chan *Transport.TextMessage)
ec.BindRecvChan("Messaging.Text.Channel", receiveChannel)

for m := range receiveChannel {
    fmt.Println("Got channel'ed message: \"", m.Body, "\" with the Id ", m.Id, ".")
}
```

Ok, that&#8217;s all in the topic of NATS. I hope you liked it and discovered something new! Please comment if you have any opinions, or don&#8217;t like something, or just want me to write about something.

Now go and build something great!

 [1]: https://hub.docker.com/_/nats/
 [2]: https://github.com/nats-io/gnatsd/releases/
 [3]: https://github.com/nats-io/gnatsd
 [4]: https://jacobmartins.com/2016/05/24/practical-golang-using-protobuffs/
 [5]: https://jacobmartins.com/2016/03/14/web-app-using-microservices-in-go-part-1-design/