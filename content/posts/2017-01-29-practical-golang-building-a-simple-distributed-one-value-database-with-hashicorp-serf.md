---
title: 'Practical Golang: Building a simple, distributed one-value database with Hashicorp Serf'
author: Jacob Martin
type: legacy-posts
date: 2017-01-29T11:24:18+00:00
url: /2017/01/29/practical-golang-building-a-simple-distributed-one-value-database-with-hashicorp-serf/
categories:
  - Go
  - Practical Golang
tags:
  - cluster
  - database
  - distributed
  - go
  - golang
  - microservice
  - microservices
  - resiliency
  - server
  - tutorial

---
## Introduction

With the advent of _distributed applications_, we see new storage solutions emerging constantly.  
They include, but are not limited to, [Cassandra][1], [Redis][2], [CockroachDB][3], [Consul][4] or [RethinkDB][5].  
Most of you probably use one, or more, of them.

They seem to be really complex systems, because they actually are. This can&#8217;t be denied.  
But it&#8217;s pretty easy to write a simple, one value database, featuring _high availability_.  
You probably wouldn&#8217;t use anything near this in production, but it should be a fruitful learning experience for you nevertheless.  
If you&#8217;re interested, read on!

## Dependencies

You&#8217;ll need to

    go get github.com/hashicorp/serf/serf
    

as a key dependency.

We&#8217;ll also use those for convenience&#8217;s sake:

    "github.com/gorilla/mux"
    "github.com/pkg/errors"
    "golang.org/x/sync/errgroup"
    

## Small overview

What will we build? We&#8217;ll build a one-value _clustered_ database. Which means, numerous instances of our application will be able to work together.  
You&#8217;ll be able to set or get the value using a REST interface. The value will then shortly be spread across the cluster using the Gossip protocol.  
Which means, every node tells a part of the cluster about the current state of the variable in set intervals. But because later each of those also tells a part of the cluster about the state, the whole cluster ends up having been informed shortly.

It&#8217;ll use Serf for easy cluster membership, which uses SWIM under the hood. SWIM is a more advanced Gossip-like algorithm, which you can read on about [here][6].

Now let&#8217;s get to the implementation&#8230;

## Getting started

First, we&#8217;ll of course have to put in all our imports:

```go
import (
    "context"
    "fmt"
    "log"
    "math/rand"
    "net/http"
    "os"
    "strconv"
    "sync"
    "time"

    "github.com/gorilla/mux"
    "github.com/hashicorp/serf/serf"
    "github.com/pkg/errors"
    "golang.org/x/sync/errgroup"
)
```

Following this, it&#8217;s time to write a simple thread-safe, one-value store.  
An important thing is, the database will also hold the _generation_ of the variable. This way, when one instance gets notified about a new value, it can check if the incoming notification actually has a higher generation count. Only then, will it change the current local value.  
So our database structure will hold exactly this: the number, generation and a mutex.

```go
type oneAndOnlyNumber struct {
    num        int
    generation int
    numMutex   sync.RWMutex
}

func InitTheNumber(val int) *oneAndOnlyNumber {
    return &oneAndOnlyNumber{
        num: val,
    }
}
```

We&#8217;ll also need a way to set and get the value.  
Setting the value will also advance the generation count, so when we notify the rest of this cluster, we will overwrite their values and generation counts.

```go
func (n *oneAndOnlyNumber) setValue(newVal int) {
    n.numMutex.Lock()
    defer n.numMutex.Unlock()
    n.num = newVal
    n.generation = n.generation + 1
}

func (n *oneAndOnlyNumber) getValue() (int, int) {
    n.numMutex.RLock()
    defer n.numMutex.RUnlock()
    return n.num, n.generation
}
```

Finally, we will need a way to notify the database of changes that happened elsewhere, if they have a higher generation count.  
For that we&#8217;ll have a small notify method, which will return true, if anything has been changed:

```go
func (n *oneAndOnlyNumber) notifyValue(curVal int, curGeneration int) bool {
    if curGeneration > n.generation {
        n.numMutex.Lock()
        defer n.numMutex.Unlock()
        n.generation = curGeneration
        n.num = curVal
        return true
    }
    return false
}
```

We&#8217;ll also create a const describing how many nodes we will notify about the new value every time.

```go
const MembersToNotify = 2
```

Now let&#8217;s get to the actual functioning of the application. First we&#8217;ll have to start an instance of serf, using two variables. The address of our instance in the network and the -optional- address of the cluster to join.

```go
func main() {
    cluster, err := setupCluster(
        os.Getenv("ADVERTISE_ADDR"),
        os.Getenv("CLUSTER_ADDR"))
    if err != nil {
        log.Fatal(err)
    }
    defer cluster.Leave()
```

How does the setupCluster function work, you may ask? Here it is:

```go
func setupCluster(advertiseAddr string, clusterAddr string) (*serf.Serf, error) {
    conf := serf.DefaultConfig()
    conf.Init()
    conf.MemberlistConfig.AdvertiseAddr = advertiseAddr

    cluster, err := serf.Create(conf)
    if err != nil {
        return nil, errors.Wrap(err, "Couldn't create cluster")
    }

    _, err = cluster.Join([]string{clusterAddr}, true)
    if err != nil {
        log.Printf("Couldn't join cluster, starting own: %v\n", err)
    }

    return cluster, nil
}
```

As we can see, we are creating the cluster, only changing the advertise address.

If the creation fails, we of course return the error.  
If the joining fails though, it means that we either didn&#8217;t get a cluster address,  
or the cluster doesn&#8217;t exist (omitting network failures), which means we can safely ignore that and just log it.

To continue with, we initialize the database and the REST API:  
(I&#8217;ve really chosen the number at random&#8230; really!)

```go
    theOneAndOnlyNumber := InitTheNumber(42)
    launchHTTPAPI(theOneAndOnlyNumber)
```

And this is what the API creation looks like:

```go
func launchHTTPAPI(db *oneAndOnlyNumber) {
    go func() {
        m := mux.NewRouter()
```

We first asynchronously start our server. Then we declare our getter:

```go
        m.HandleFunc("/get", func(w http.ResponseWriter, r *http.Request) {
            val, _ := db.getValue()
            fmt.Fprintf(w, "%v", val)
        })
```

our setter:

```go
m.HandleFunc("/set/{newVal}", func(w http.ResponseWriter, r *http.Request) {
            vars := mux.Vars(r)
            newVal, err := strconv.Atoi(vars["newVal"])
            if err != nil {
                w.WriteHeader(http.StatusBadRequest)
                fmt.Fprintf(w, "%v", err)
                return
            }

            db.setValue(newVal)

            fmt.Fprintf(w, "%v", newVal)
        })
```

and finally the API endpoint which allows other _nodes_ to notify this instance of changes:

```go
        m.HandleFunc("/notify/{curVal}/{curGeneration}", func(w http.ResponseWriter, r *http.Request) {
            vars := mux.Vars(r)
            curVal, err := strconv.Atoi(vars["curVal"])
            if err != nil {
                w.WriteHeader(http.StatusBadRequest)
                fmt.Fprintf(w, "%v", err)
                return
            }
            curGeneration, err := strconv.Atoi(vars["curGeneration"])
            if err != nil {
                w.WriteHeader(http.StatusBadRequest)
                fmt.Fprintf(w, "%v", err)
                return
            }

            if changed := db.notifyValue(curVal, curGeneration); changed {
                log.Printf(
                    "NewVal: %v Gen: %v Notifier: %v",
                    curVal,
                    curGeneration,
                    r.URL.Query().Get("notifier"))
            }
            w.WriteHeader(http.StatusOK)
        })
        log.Fatal(http.ListenAndServe(":8080", m))
    }()
}
```

It&#8217;s also here where we start our server and print some debug info when getting notified of new values by other _members_ of our cluster.

* * *

Great, we&#8217;ve got a way to talk to our service now. Time to make it actually spread all the information.  
We&#8217;ll also be printing debug info regularly.

To begin with, let&#8217;s initiate our _context_ (that&#8217;s always a good idea in the main function).  
We&#8217;ll also put a value into it, the name of our host, just for the debug logs.  
It&#8217;s a good thing to put into the context, as it&#8217;s not something crucial for the functioning of our program,  
and the context will get passed further anyways.

```go
    ctx := context.Background()
    if name, err := os.Hostname(); err == nil {
        ctx = context.WithValue(ctx, "name", name)
    }
```

Having done this, we can set up our main loop, including the intervals at which we&#8217;ll be sending state updates to peers and printing debug info.

```go
    debugDataPrinterTicker := time.Tick(time.Second * 5)
    numberBroadcastTicker := time.Tick(time.Second * 2)
    for {
        select {
        case <-numberBroadcastTicker:
        // Notification code goes here...
        case <-debugDataPrinterTicker:
            log.Printf("Members: %v\n", cluster.Members())

            curVal, curGen := theOneAndOnlyNumber.getValue()
            log.Printf("State: Val: %v Gen: %v\n", curVal, curGen)
        }
    }
```

Ok, that seems to be it.

Just kidding. Time to finish up our service with the notification code.  
We&#8217;ll now get a list of **other** members in the cluster, set a timeout, and asynchronously notify a part of those others.

```go
        case <-numberBroadcastTicker:
            members := getOtherMembers(cluster)

            ctx, _ := context.WithTimeout(ctx, time.Second*2)
            go notifyOthers(ctx, members, theOneAndOnlyNumber)
```

Now, let&#8217;s look at the _getOtherMembers_ function. It&#8217;s actually just a function scanning through the memberlist, deleting ourselves and other nodes that aren&#8217;t alive at the moment.

```go
func getOtherMembers(cluster *serf.Serf) []serf.Member {
    members := cluster.Members()
    for i := 0; i < len(members); {
        if members[i].Name == cluster.LocalMember().Name || members[i].Status != serf.StatusAlive {
            if i < len(members)-1 {
                members = append(members[:i], members[i + 1:]...)
            } else {
                members = members[:i]
            }
        } else {
            i++
        }
    }
    return members
}
```

There&#8217;s not much to it I suppose. It&#8217;s using slicing to cut out or cut off members not conforming to our predicates.

Finally the function we use to notify others:

```go
func notifyOthers(ctx context.Context, otherMembers []serf.Member, db *oneAndOnlyNumber) {
    g, ctx := errgroup.WithContext(ctx)

    if len(otherMembers) <= MembersToNotify {
        for _, member := range otherMembers {
            curMember := member
            g.Go(func() error {
                return notifyMember(ctx, curMember.Addr.String(), db)
            })
        }
    } else {
        randIndex := rand.Int() % len(otherMembers)
        for i := 0; i < MembersToNotify; i++ {
            g.Go(func() error {
                return notifyMember(
                    ctx,
                    otherMembers[(randIndex + i) % len(otherMembers)].Addr.String(),
                    db)
            })
        }
    }

    err := g.Wait()
    if err != nil {
        log.Printf("Error when notifying other members: %v", err)
    }
}
```

If there are only two members then it sends the notifications to them, otherwise it chooses a random index in the members array and chooses subsequent members from there on.  
How does the errgroup work? It&#8217;s a nifty library Brian Ketelsen wrote a [great article][7] about. It&#8217;s basically a wait group which also gathers errors and aborts when one happens.

Now to finish our code, the _notifyMember_ function:

```go
func notifyMember(ctx context.Context, addr string, db *oneAndOnlyNumber) error {
    val, gen := db.getValue()
    req, err := http.NewRequest("POST", fmt.Sprintf("http://%v:8080/notify/%v/%v?notifier=%v", addr, val, gen, ctx.Value("name")), nil)
    if err != nil {
        return errors.Wrap(err, "Couldn't create request")
    }
    req = req.WithContext(ctx)

    _, err = http.DefaultClient.Do(req)
    if err != nil {
        return errors.Wrap(err, "Couldn't make request")
    }
    return nil
}

```

We craft a path with the formula _{nodeAddress}:8080/notify/{curVal}/{curGen}?notifier={selfHostName}_  
We add the context to the request, so we get the timeout functionality, and finally make the request.

And that&#8217;s actually all there is to the code.

## Testing our database

We&#8217;ll test our database using docker. The necessary dockerfile to put into your project directory looks like this:

```go
FROM alpine
WORKDIR /app
COPY distApp /app/
ENTRYPOINT ["./distApp"]
```

Now, first build your application (if you&#8217;re not on linux, you have to set the env variables GOOS=linux and GOARCH=amd64)  
Later build the docker image:

    docker build -t distapp .
    

And finally we can launch it. To supply the necessary environment variables, we&#8217;ll need to know what ip address the containers will get.  
First run:

    docker network inspect bridge
    

Bridge is the default network containers get assigned to. You should get something like this:

    [
        {
            "Name": "bridge",
            "Id": "b56a19697ed9d30488f189d5517fd79f04a4df70c8bbc07d8f3c49a491f10433",
            "Created": "2017-01-29T10:48:05.1592086Z",
            "Scope": "local",
            "Driver": "bridge",
            "EnableIPv6": false,
            "IPAM": {
                "Driver": "default",
                "Options": null,
                "Config": [
                    {
                        "Subnet": "172.17.0.0/16",
                        "Gateway": "172.17.0.1" <-- this is what we need
                    }
                ]
            },
            "Internal": false,
            "Attachable": false,
            "Containers": {},
            "Options": {
                "com.docker.network.bridge.default_bridge": "true",
                "com.docker.network.bridge.enable_icc": "true",
                "com.docker.network.bridge.enable_ip_masquerade": "true",
                "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
                "com.docker.network.bridge.name": "docker0",
                "com.docker.network.driver.mtu": "1500"
            },
            "Labels": {}
        }
    ]
    

What&#8217;s important for us is the gateway. In this case, our containers would be spawned with IP addresses from 172.17.0.2

So now we can start a few containers:

    docker run -e ADVERTISE_ADDR=172.17.0.2 -p 8080:8080 distapp
    docker run -e ADVERTISE_ADDR=172.17.0.3 -e CLUSTER_ADDR=172.17.0.2 -p 8081:8080 distapp
    docker run -e ADVERTISE_ADDR=172.17.0.4 -e CLUSTER_ADDR=172.17.0.3 -p 8082:8080 distapp
    docker run -e ADVERTISE_ADDR=172.17.0.5 -e CLUSTER_ADDR=172.17.0.4 -p 8083:8080 distapp
    

Next on you can test your deployment by stopping and starting containers, and setting/getting the variables at:

    localhost:8080/set/5
    localhost:8082/get/5
    etc...
    

## Conclusion

What&#8217;s important, this is a really basic distributed system, it may become inconsistent (if you update the value on two different machines simultaneously, the cluster will have two values depending on the machine).  
If you want to learn more, read about CAP, consensus, Paxos, RAFT, gossip, and data replication, they are all very interesting topics (at least in my opinion).

Anyways, I hope you had fun creating a small distributed system and encourage you to build your own, more advanced one, it&#8217;ll be a great learning experience for sure!

The whole code is available on [my Github][8].

 [1]: https://cassandra.apache.org/
 [2]: https://redis.io/
 [3]: https://www.cockroachlabs.com/
 [4]: https://www.consul.io/
 [5]: https://www.rethinkdb.com/
 [6]: https://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf
 [7]: https://www.oreilly.com/learning/run-strikingly-fast-parallel-file-searches-in-go-with-sync-errgroup
 [8]: https://github.com/cube2222/Blog/tree/master/Building%20a%20simple%20distributed%20database