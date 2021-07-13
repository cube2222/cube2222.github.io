---
title: 'Practical Golang: Event multicast/subscription service'
author: Jacob Martin
type: legacy-posts
date: 2016-04-04T10:30:24+00:00
url: /2016/04/04/practical-golang-event-multicastsubscription-service/
categories:
  - Practical Golang
tags:
  - architecture
  - event
  - go
  - golang
  - message
  - microservice
  - subscription

---
## Introduction

In our microservice architectures we always need a method for communicating between services. There are various ways to achieve this. Few of them are, but are not limited to: _Remote Procedure Call_, _REST API&#8217;s_, _message BUSses_. In this comprehensive tutorial we&#8217;ll write a service, which you can use to distribute messages/events across your system.

### Design

How will it work? It will accept registering subscribers (other microservices). Whenever it gets a message from a microservice, it will send it further to all subscribers, using a REST call to the other microservices /event URL.

Subscribers will need to call a _keep-alive_ URL regularly, otherwise they will get removed from the subscriber list. This protects us from sending messages to too many ghost subscribers.

## Implementation

Let&#8217;s start with a basic structure. We&#8217;ll define the **API** and set up our two main data structures:  
1. The **_subscriber list_** with their register/lastKeepAlive dates.  
2. The **_mutex_** controlling access to our subscriber list.

```go
package main

import (
    "net/http"
    "time"
    "sync"
    "fmt"
    "net/url"
    "io/ioutil"
    "bytes"
)

var registeredServiceStorage map[string]time.Time
var serviceStorageMutex sync.RWMutex

func main() {
    registeredServiceStorage = make(map[string]time.Time)
    serviceStorageMutex = sync.RWMutex{}

    http.HandleFunc("/registerAndKeepAlive", registerAndKeepAlive)
    http.HandleFunc("/deregister", deregister)
    http.HandleFunc("/sendMessage", handleMessage)
    http.HandleFunc("/listSubscribers", handleSubscriberListing)

    go killZombieServices()
    http.ListenAndServe(":3000", nil)
}

func registerAndKeepAlive(w http.ResponseWriter, r *http.Request) {
}

func deregister(w http.ResponseWriter, r *http.Request) {
}

func handleMessage(w http.ResponseWriter, r *http.Request) {
}

func sendMessageToSubscriber(data []byte, address string) {
}

func handleSubscriberListing(w http.ResponseWriter, r *http.Request) {
}

func killZombieServices() {
}
```

We initialize our subscriber list and mutex, and also launch, on another thread, a function that will regularly delete _ghost subscribers_.

So far so good!  
We can now start getting into each functions implementation.

We can begin with the registerAndKeepAlive which does both things. Registering a new subscriber, or updating an existing one. This works because in both cases we just update the map entry with the subscriber address to contain the current time.

```go
func registerAndKeepAlive(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodPost {
        //Subscriber registration
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only POST accepted")
    }
}
```

The register function should be called with a **POST** request. That&#8217;s why the first thing we do, is checking if the method is right, otherwise we answer with an error. If it&#8217;s ok, then we register the client:

```go
if r.Method == http.MethodPost {
    values, err := url.ParseQuery(r.URL.RawQuery)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error:", err)
        return
    }
    if len(values.Get("address")) == 0 {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error:","Wrong input address.")
        return
    }

}
```

We check if the _URL arguments_ are correct, and finally register the subscriber:

```go
if len(values.Get("address")) == 0 {
    w.WriteHeader(http.StatusBadRequest)
    fmt.Fprint(w, "Error:","Wrong input address.")
    return
}

serviceStorageMutex.Lock()
registeredServiceStorage[values.Get("address")] = time.Now()
serviceStorageMutex.Unlock()

fmt.Fprint(w, "success")
```

Awesome!

Let&#8217;s now implement the function which shall delete the entry when the subscriber wants to deregister.

```go
func deregister(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodDelete {
        values, err := url.ParseQuery(r.URL.RawQuery)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", err)
            return
        }
        if len(values.Get("address")) == 0 {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:","Wrong input address.")
            return
        }

        //Subscriber deletion will come here

    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only DELETE accepted")
    }
}
```

Again do we check if the _request method_ is good and if the _address_ argument is correct. If that&#8217;s the case, then we can remove this client from our subscriber list.

```go
if len(values.Get("address")) == 0 {
    w.WriteHeader(http.StatusBadRequest)
    fmt.Fprint(w, "Error:","Wrong input address.")
    return
}

serviceStorageMutex.Lock()
delete(registeredServiceStorage, values.Get("address"))
serviceStorageMutex.Unlock()

fmt.Fprint(w, "success")
```

Now it&#8217;s time for the main functionality. Namely handling messages and sending them to all subscribers:

```go
func handleMessage(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodPost {

    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only POST accepted")
    }
}
```

As usual, we check if the request method is correct.

Then, we read the data we got, so we can pass it to multiple concurrent sending functions.

```go
if r.Method == http.MethodPost {

    data, err := ioutil.ReadAll(r.Body)
    if err != nil {
        fmt.Println(err)
    }
    //...
}
```

We then lock the mutex **_for read_**. That&#8217;s important so that we can handle huge amounts of messages efficiently. Basically, it means that we allow others to read while we are reading, because concurrent reading is supported by maps. We can use this unless there&#8217;s no one modifying the map.

While we lock the map for read, we check the list of subscribers we have to send the message to, and start concurrent functions that will do the sending. As we don&#8217;t want to lock the map for the entire sending time, we only need the addresses.

```go
    data, err := ioutil.ReadAll(r.Body)
    if err != nil {
        fmt.Println(err)
    }

    serviceStorageMutex.RLock()
    for address, _ := range registeredServiceStorage {
        go sendMessageToSubscriber(data, address)
    }
    serviceStorageMutex.RUnlock()

    fmt.Fprint(w, "success")
```

Which means we now have to implement the _sendMessageToSubscriber(&#8230;)_ function.

It&#8217;s pretty simple, we just make a post, and print an error if it happened.

```go
func sendMessageToSubscriber(data []byte, address string) {
    _, err := http.Post("http://" + address + "/event", "", bytes.NewBuffer(data))
    if err != nil {
        fmt.Println(err)
    }
}
```

It&#8217;s important to notice, that we have to create a _buffer_ from the data, as the _http.Post(&#8230;)_ function needs a reader type data structure.

We&#8217;ll also implement the function which makes it possible to list all the subscribers. Mainly for debugging purposes. There&#8217;s nothing new in it. We check if the method is alright, lock the mutex for read, and finally print the map with a correct format of the register time.

```go
func handleSubscriberListing(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodGet {
        serviceStorageMutex.RLock()

        for address, registerTime := range registeredServiceStorage {
            fmt.Fprintln(w, address, " : ", registerTime.Format(time.RFC3339))
        }

        serviceStorageMutex.RUnlock()
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only GET accepted")
    }
}
```

Now there&#8217;s only one function left. The one that will make sure no ghost services stay for too long. It will check all the services once per minute. This way we&#8217;re making it cheap on performance:

```go
func killZombieServices() {
    t := time.Tick(1 * time.Minute)

    for range t {
    }
}
```

This is a nice way to launch the code every minute. We create a channel which will send us the time every minute, and range over it, ignoring the received values.

We can now get the check and remove working.

```go
for range t {
    timeNow := time.Now()
    serviceStorageMutex.Lock()
    for address, timeKeepAlive := range registeredServiceStorage {
        if timeNow.Sub(timeKeepAlive).Minutes() > 2 {
            delete(registeredServiceStorage, address)
        }
    }
    serviceStorageMutex.Unlock()
}
```

We just range over the subscribers and delete those that haven&#8217;t kept their subscription alive.

To add to that, if you wanted you could first make a read-only pass over the subscribers, and immediately after that, make a **write-locked** deletion of the ones you found. This would allow others to read the map while you&#8217;re finding subscribers to delete.

## Conclusion

That&#8217;s all! Have fun with creating an infrastructure based on such a service!