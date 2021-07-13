---
title: 'Web app using Microservices in Go: Part 2 – k/v store and Database'
author: Jacob Martin
type: legacy-posts
date: 2016-03-16T12:30:40+00:00
url: /2016/03/16/web-app-using-microservices-in-go-part-2-kv-store-and-database/
categories:
  - Go
tags:
  - architecture
  - database
  - go
  - golang
  - k/v store
  - key value store
  - microservice
  - web app

---
[Previous part][1]

## Introduction

In this part we will implement part of the microservices needed for our web app. We will implement the:  
* key-value store  
* Database

This will be a pretty code heavy tutorial so concentrate and have fun!

## The key-value store

### Design

The design hasn&#8217;t changed much. We will save the key-value pairs as a global map, and create a global mutex for concurrent access. We&#8217;ll also add the ability to list all key-value pairs for debugging/analytical purposes. We will also add the ability to delete existing entries.

First, let&#8217;s create the structure:

<pre><code class="go">package main

import (
    "net/http"
    "sync"
    "net/url"
    "fmt"
)

var keyValueStore map[string]string
var kVStoreMutex sync.RWMutex

func main() {
    keyValueStore = make(map[string]string)
    kVStoreMutex = sync.RWMutex{}
    http.HandleFunc("/get", get)
    http.HandleFunc("/set", set)
    http.HandleFunc("/remove", remove)
    http.HandleFunc("/list", list)
    http.ListenAndServe(":3000", nil)
}

func get(w http.ResponseWriter, r *http.Request) {
}

func set(w http.ResponseWriter, r *http.Request) {
}

func remove(w http.ResponseWriter, r *http.Request) {
}

func list(w http.ResponseWriter, r *http.Request) {
}

</code></pre>

And now let&#8217;s dive into the implementation.

First, we should add parameter parsing in the get function and verify that the key parameter is right.

<pre><code class="go">func get(w http.ResponseWriter, r *http.Request) {
    if(r.Method == http.MethodGet) {
        values, err := url.ParseQuery(r.URL.RawQuery)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", err)
            return
        }
        if len(values.Get("key")) == 0 {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:","Wrong input key.")
            return
        }
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only GET accepted.")
    }
}
</code></pre>

The _key_ shouldn&#8217;t have a length of 0, hence the length check. We also check if the method is GET, if it isn&#8217;t we print it and set the status code to **_bad request_**.  
We answer with an explicit **_Error:_** before each error message so it doesn&#8217;t get misinterpreted by the client as a value.

Now, let&#8217;s access our map and send back a response:

<pre><code class="go">if len(values.Get("key")) == 0 {
    w.WriteHeader(http.StatusBadRequest)
    fmt.Fprint(w, "Error:","Wrong input key.")
    return
}

kVStoreMutex.RLock()
value := keyValueStore[string(values.Get("key"))]
kVStoreMutex.RUnlock()

fmt.Fprint(w, value)
</code></pre>

We copy the value into a variable so that we don&#8217;t block the map while sending back the response.

Now let&#8217;s create the set function, it&#8217;s actually pretty similar.

<pre><code class="go">func set(w http.ResponseWriter, r *http.Request) {
    if(r.Method == http.MethodPost) {
        values, err := url.ParseQuery(r.URL.RawQuery)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", err)
            return
        }
        if len(values.Get("key")) == 0 {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", "Wrong input key.")
            return
        }
        if len(values.Get("value")) == 0 {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", "Wrong input value.")
            return
        }

        kVStoreMutex.Lock()
        keyValueStore[string(values.Get("key"))] = string(values.Get("value"))
        kVStoreMutex.Unlock()

        fmt.Fprint(w, "success")
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only POST accepted.")
    }
}
</code></pre>

The only difference is that we also check if there is a right value parameter and check if the method is POST.

Now we can add the implementation of the list function which is also pretty simple:

<pre><code class="go">func list(w http.ResponseWriter, r *http.Request) {
    if(r.Method == http.MethodGet) {
        kVStoreMutex.RLock()
        for key, value := range keyValueStore {
            fmt.Fprintln(w, key, ":", value)
        }
        kVStoreMutex.RUnlock()
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only GET accepted.")
    }
}
</code></pre>

It just ranges over the map and prints everything. Simple yet effective.

And to finish the key-value store we will implement the _remove_ function:

<pre><code class="go">func remove(w http.ResponseWriter, r *http.Request) {
    if(r.Method == http.MethodDelete) {
        values, err := url.ParseQuery(r.URL.RawQuery)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", err)
            return
        }
        if len(values.Get("key")) == 0 {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", "Wrong input key.")
            return
        }

        kVStoreMutex.Lock()
        delete(keyValueStore, values.Get("key"))
        kVStoreMutex.Unlock()

        fmt.Fprint(w, "success")
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only DELETE accepted.")
    }
}
</code></pre>

It&#8217;s the same as setting a value, but instead of setting it we delete it.

## The database

### Design

After thinking through the design, I decided that it would be better if the database generated the task _Id_&#8216;s. This will also make it easier to get the last non-finished task and generate consecutive _Id_&#8216;s

How it will work:  
* It will save new tasks assigning consecutive _Id_&#8216;s.  
* It will allow to get a new task to do.  
* It will allow to get a task by _Id_.  
* It will allow to set a task by _Id_.  
* The state will be represented by an int:  
* 0 &#8211; not started  
* 1 &#8211; in progress  
* 2 &#8211; finished  
* It will change the state of a task to _not started_ if it&#8217;s been too long _in progress_. (maybe someone started to work on it but has crashed)  
* It will allow to list all tasks for debugging/analytical purposes.

![Database microservice post][2] 

### Implementation

First, we should create the API and later we will add the implementations of the functionality as before with the key-value store. We will also need a global map being our data store, a variable pointing to the oldest not started task, and mutexes for accessing the datastore and pointer.

<pre><code class="go">package main

import (
    "net/http"
    "net/url"
    "fmt"
)

type Task struct {
}

var datastore map[int]Task
var datastoreMutex sync.RWMutex
var oldestNotFinishedTask int // remember to account for potential int overflow in production. Use something bigger.
var oNFTMutex sync.RWMutex

func main() {

    datastore = make(map[int]Task)
    datastoreMutex = sync.RWMutex{}
    oldestNotFinishedTask = 0
    oNFTMutex = sync.RWMutex{}

    http.HandleFunc("/getById", getById)
    http.HandleFunc("/newTask", newTask)
    http.HandleFunc("/getNewTask", getNewTask)
    http.HandleFunc("/finishTask", finishTask)
    http.HandleFunc("/setById", setById)
    http.HandleFunc("/list", list)
    http.ListenAndServe(":3001", nil)
}

func getById(w http.ResponseWriter, r *http.Request) {
}

func newTask(w http.ResponseWriter, r *http.Request) {
}

func getNewTask(w http.ResponseWriter, r *http.Request) {
}

func finishTask(w http.ResponseWriter, r *http.Request) {
}

func setById(w http.ResponseWriter, r *http.Request) {
}

func list(w http.ResponseWriter, r *http.Request) {
}
</code></pre>

We also already declared the **_Task_** type which we will use for storage.

So far so good. Now let&#8217;s implement all those functions!

First, let&#8217;s implement the getById function.

<pre><code class="go">func getById(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodGet {
        values, err := url.ParseQuery(r.URL.RawQuery)
        if err != nil {
            fmt.Fprint(w, err)
            return
        }
        if len(values.Get("id")) == 0 {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Wrong input")
            return
        }

        id, err := strconv.Atoi(string(values.Get("id")))
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, err)
            return
        }

        datastoreMutex.RLock()
        bIsInError := err != nil || id &gt;= len(datastore) // Reading the length of a slice must be done in a synchronized manner. That's why the mutex is used.
        datastoreMutex.RUnlock()

        if bIsInError {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Wrong input")
            return
        }

        datastoreMutex.RLock()
        value := datastore[id]
        datastoreMutex.RUnlock()

        response, err := json.Marshal(value)

        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, err)
            return
        }

        fmt.Fprint(w, string(response))
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only GET accepted")
    }
}
</code></pre>

We check if the **_GET_** method has been used. Later we parse the _id_ argument and check if it&#8217;s proper. We then get the _id_ as an **int** using the _strconv.Atoi_ function. Next we make sure it is not out of bounds for our _datastore_, which we have to do using _mutexes_ because we&#8217;re accessing a map which could be accessed from another thread. If everything is ok, then, again using _mutexes_, we get the task using the _id_.

After that we use the _JSON_ library to marshal our struct into a _JSON object_ and if that finishes without problems we send the _JSON object_ to the client.

It&#8217;s also time to implement our _Task_ struct:

<pre><code class="go">type Task struct {
    Id int `json:"id"`
    State int `json:"state"`
}
</code></pre>

It&#8217;s all that&#8217;s needed. We also added the information the _JSON_ marshaller needs.

We can now go on with implementing the _newTask_ function:

<pre><code class="go">func newTask(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodPost {
        datastoreMutex.Lock()
        taskToAdd := Task{
            Id: len(datastore),
            State: 0,
        }
        datastore[taskToAdd.Id] = taskToAdd
        datastoreMutex.Unlock()

        fmt.Fprint(w, taskToAdd.Id)
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only POST accepted")
    }
}
</code></pre>

It&#8217;s pretty small actually. Creating a new _Task_ with the next id and adding it to the _datastore_. After that it sends back the new _Tasks_ Id.

That means we can go on to implementing the function used to list all _Tasks_, as this helps with debugging during writing.

It&#8217;s basically the same as with the key-value store:

<pre><code class="go">func list(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodGet {
        datastoreMutex.RLock()
        for key, value := range datastore {
            fmt.Fprintln(w, key, ": ", "id:", value.Id, " state:", value.State)
        }
        datastoreMutex.RUnlock()
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only GET accepted")
    }
}
</code></pre>

Ok, so now we will implement the function which can set the _Task_ by _id_:

<pre><code class="go">func setById(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodPost {
        taskToSet := Task{}

        data, err := ioutil.ReadAll(r.Body)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, err)
            return
        }
        err = json.Unmarshal([]byte(data), &taskToSet)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, err)
            return
        }

        bErrored := false
        datastoreMutex.Lock()
        if taskToSet.Id &gt;= len(datastore) || taskToSet.State &gt; 2 || taskToSet.State &lt; 0 {
            bErrored = true
        } else {
            datastore[taskToSet.Id] = taskToSet
        }
        datastoreMutex.Unlock()

        if bErrored {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error: Wrong input")
            return
        }

        fmt.Fprint(w, "success")
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only POST accepted")
    }
}
</code></pre>

Nothing new. We get the request and try to unmarshal it. If it succeeds we put it into the map, checking if it isn&#8217;t out of bounds or if the state is invalid. If it is then we print an error, otherwise we print _success_.

If we already have this we can now implement the finish task function, because it&#8217;s very simple:

<pre><code class="go">func finishTask(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodPost {
        values, err := url.ParseQuery(r.URL.RawQuery)
        if err != nil {
            fmt.Fprint(w, err)
            return
        }
        if len(values.Get("id")) == 0 {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Wrong input")
            return
        }

        id, err := strconv.Atoi(string(values.Get("id")))

        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, err)
            return
        }

        updatedTask := Task{Id: id, State: 2}

        bErrored := false

        datastoreMutex.Lock()
        if datastore[id].State == 1 {
            datastore[id] = updatedTask
        } else {
            bErrored = true
        }
        datastoreMutex.Unlock()

        if bErrored {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error: Wrong input")
            return
        }

        fmt.Fprint(w, "success")
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only POST accepted")
    }
}
</code></pre>

It&#8217;s pretty similar to the _getById_ function. The difference here is that here we update the state and only if it is currently _in progress_.

And now to one of the most interesting functions. The _getNewTask_ function. It has to handle updating the oldest known finished task, and it also needs to handle the situation when someone takes a task but crashes during work. This would lead to a ghost task forever being _in progress_. That&#8217;s why we&#8217;ll add functionality which after 120 seconds from starting a task will set it back to _not started_:

<pre><code class="go">func getNewTask(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodPost {

        bErrored := false

        datastoreMutex.RLock()
        if len(datastore) == 0 {
            bErrored = true
        }
        datastoreMutex.RUnlock()

        if bErrored {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error: No non-started task.")
            return
        }

        taskToSend := Task{Id: -1, State: 0}

        oNFTMutex.Lock()
        datastoreMutex.Lock()
        for i := oldestNotFinishedTask; i &lt; len(datastore); i++ {
            if datastore[i].State == 2 && i == oldestNotFinishedTask {
                oldestNotFinishedTask++
                continue
            }
            if datastore[i].State == 0 {
                datastore[i] = Task{Id: i, State: 1}
                taskToSend = datastore[i]
                break
            }
        }
        datastoreMutex.Unlock()
        oNFTMutex.Unlock()

        if taskToSend.Id == -1 {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error: No non-started task.")
            return
        }

        myId := taskToSend.Id

        go func() {
            time.Sleep(time.Second * 120)
            datastoreMutex.Lock()
            if datastore[myId].State == 1 {
                datastore[myId] = Task{Id: myId, State: 0}
            }
            datastoreMutex.Unlock()
        }()

        response, err := json.Marshal(taskToSend)

        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, err)
            return
        }

        fmt.Fprint(w, string(response))
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only POST accepted")
    }
}
</code></pre>

First we try to find the oldest task that hasn&#8217;t started yet. By the way we update the oldestNotFinishedTask variable. If a task is finished and is pointed on by the variable, the variable get&#8217;s incremented. If we find something that&#8217;s not started, then we break out of the loop and send it back to the user setting it to _in progress_. However, on the way we start a function on another thread that will change the state of the task back to _not started_ if it&#8217;s still in progress after 120 seconds.

Now the last thing. A database is useless&#8230; when you don&#8217;t know where it is! That&#8217;s why we&#8217;ll now implement the mechanism that the database will use to register itself in the _key-value store_:

<pre><code class="go">func main() {

    if !registerInKVStore() {
        return
    }

    datastore = make(map[int]Task)
</code></pre>

and later we define the function:

<pre><code class="go">func registerInKVStore() bool {
    if len(os.Args) &lt; 3 {
        fmt.Println("Error: Too few arguments.")
        return false
    }
    databaseAddress := os.Args[1] // The address of itself
    keyValueStoreAddress := os.Args[2]

    response, err := http.Post("http://" + keyValueStoreAddress + "/set?key=databaseAddress&value=" + databaseAddress, "", nil)
    if err != nil {
        fmt.Println(err)
        return false
    }
    data, err := ioutil.ReadAll(response.Body)
    if err != nil {
        fmt.Println(err)
        return false
    }
    if response.StatusCode != http.StatusOK {
        fmt.Println("Error: Failure when contacting key-value store: ", string(data))
        return false
    }
    return true
}
</code></pre>

We check if there are at least 3 arguments. (The first being the executable) We read the current _database address_ from the second argument and the _key-value store address_ from the third argument. We use them to make a POST request where we add a **_databaseAddress_** key to the _k/v store_ and set its value to the current _database address_. If the status code of the response isn&#8217;t **_OK_** then we know we messed up and we print the error we got. After that we quit the program.

## Conclusion

We now have finished our _k/v store_ and our _database_. You can even test them now using a REST client. (I used [this one][3].) Remember that the code is subject to change if it will be necessary but I don&#8217;t think so. I hope you enjoyed the tutorial! I encourage you to comment, and if you have an opposing view to mine please make sure to express it in a comment too!

**_UPDATE_**: I changed the sync.Mutex to sync.RWMutex, and in the places where we only read data I changed mutex.Lock/Unlock to mutex.RLock/RUnlock.

**_UPDATE2_**: For some reason I used a slice for the database code although I tested with a map. Sorry for that, corrected it already.

[Next part][4]

 [1]: https://jacobmartins.com/2016/03/14/web-app-using-microservices-in-go-part-1-design/
 [2]: https://www.lucidchart.com/publicSegments/view/4cf0690e-3dbb-42d9-befd-4a6efaaf6f72/image.png
 [3]: https://chrome.google.com/webstore/detail/advanced-rest-client/hgmloofddffdnphfgcellkdfbfbjeloo
 [4]: https://jacobmartins.com/2016/03/21/web-app-using-microservices-in-go-part-3-storage-and-master/