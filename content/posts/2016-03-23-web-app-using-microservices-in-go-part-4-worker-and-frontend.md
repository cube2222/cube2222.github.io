---
title: 'Web app using Microservices in Go: Part 4 – Worker and Frontend'
author: Jacob Martin
type: legacy-posts
date: 2016-03-23T13:34:03+00:00
url: /2016/03/23/web-app-using-microservices-in-go-part-4-worker-and-frontend/
categories:
  - Go
tags:
  - architecture
  - design
  - frontend
  - go
  - golang
  - microservice
  - tutorial
  - worker

---
[Previous part][1]

## Introduction

In this part we will finally finish writing our application. We will implement the last two services:  
1. The Worker  
2. The Frontend

## The Worker

The worker will communicate with the _Master_ to get new _Tasks_. When it gets a _Task_ it will get the corresponding data from the storage and will start working on the task. When it finishes it will send the finished data to the storage service, and if that succeeds it will register the _Task_ as finished to the _Master_.

That means that you can easily scale the workforce, by turning on additional machines, with the worker service on them. Easy scaling is good!

### Implementation

As usual we will start with a basic structure which is similar to the structure of our previous services. Although there is one big difference. There won&#8217;t be any API here as the worker will be a client. You could, if you wanted, add an API for debugging purposes. Things like getting the processor usage. But this can also be implemented using 3rd party health checking services.

So here&#8217;s our basic structure:

```go
package main

import (
    "os"
    "fmt"
    "net/http"
    "io/ioutil"
    "encoding/json"
    "time"
    "strconv"
    "image"
    "image/png"
    "image/color"
    "bytes"
    "sync"
)

type Task struct {
    Id int `json:"id"`
    State int `json:"state"`
}

var masterLocation string
var storageLocation string
var keyValueStoreAddress string

func main() {
    if len(os.Args) < 3 {
        fmt.Println("Error: Too few arguments.")
        return
    }
    keyValueStoreAddress = os.Args[1]

    response, err := http.Get("http://" + keyValueStoreAddress + "/get?key=masterAddress")
    if response.StatusCode != http.StatusOK {
        fmt.Println("Error: can't get master address.")
        fmt.Println(response.Body)
        return
    }
    data, err := ioutil.ReadAll(response.Body)
    if err != nil {
        fmt.Println(err)
        return
    }
    masterLocation = string(data)
    if len(masterLocation) == 0 {
        fmt.Println("Error: can't get master address. Length is zero.")
        return
    }

    response, err = http.Get("http://" + keyValueStoreAddress + "/get?key=storageAddress")
    if response.StatusCode != http.StatusOK {
        fmt.Println("Error: can't get storage address.")
        fmt.Println(response.Body)
        return
    }
    data, err = ioutil.ReadAll(response.Body)
    if err != nil {
        fmt.Println(err)
        return
    }
    storageLocation = string(data)
    if len(storageLocation) == 0 {
        fmt.Println("Error: can't get storage address. Length is zero.")
        return
    }
}

func getNewTask(masterAddress string) (Task, error) {
}
func getImageFromStorage(storageAddress string, myTask Task) (image.Image, error) {
}
func doWorkOnImage(myImage image.Image) image.Image {
}
func sendImageToStorage(storageAddress string, myTask Task, myImage image.Image) error {
}
func registerFinishedTask(masterAddress string, myTask Task) error {
}
```

It may seem to be much but there is nothing new actually in the most dense parts. We define the _Task_ structure first and declare the needed addresses/locations.

Later we check if there are enough arguments passed to the application, and, as we did in the previous parts, we get the addresses of the _Master_ and the _Storage_. That&#8217;s basically all.

For the worker we will want a command line parameter to set the number of concurrent threads. That&#8217;s why we check if there are 3 Arguments.

Now let&#8217;s parse the thread count:

```go
storageLocation = string(data)
if len(storageLocation) == 0 {
    fmt.Println("Error: can't get storage address. Length is zero.")
    return
}

threadCount, err := strconv.Atoi(os.Args[2])
if err != nil {
    fmt.Println("Error: Couldn't parse thread count.")
    return
}
```

We use the _atoi_ function to parse the string argument to an int.

And now we come to the main for loop which runs on each thread:

```go
myWG := sync.WaitGroup{}
myWG.Add(threadCount)
for i := 0; i < threadCount; i++ {
    go func() {
        for {
            //Do work...
        }
    }()
}
myWG.Wait()
```

Ok, what are we doing there? We create a **_waitGroup_**. The main function has to be waiting for the **goroutines** and not just finish execution, that&#8217;s why we create a waitGroup and add the thread count. You could add a functionality to break the endless for loop and after that use the _Done()_ function on the waitgroup. We won&#8217;t be adding this as we just want endless for work loops.

Now we will write down the execution process for each _Task_.  
First we get a new _Task_:

```go
for {
    myTask, err := getNewTask(masterLocation)
    if err != nil {
        fmt.Println(err)
        fmt.Println("Waiting 2 second timeout...")
        time.Sleep(time.Second * 2)
        continue
    }
```

If we error, then we wait 2 seconds, so it doesn&#8217;t make a lot of errored requests to the master at once. As this could flood the other services.

If we successfully get the _Task_, then we get the image to work on from the storage:

```go
myTask, err := getNewTask(masterLocation)
if err != nil {
    fmt.Println(err)
    fmt.Println("Waiting 2 second timeout...")
    time.Sleep(time.Second * 2)
    continue
}

myImage, err := getImageFromStorage(storageLocation, myTask)
if err != nil {
    fmt.Println(err)
    fmt.Println("Waiting 2 second timeout...")
    time.Sleep(time.Second * 2)
    continue
}
```

Ok, now it&#8217;s time to do something with the image!!!

```go
myImage, err := getImageFromStorage(storageLocation, myTask)
if err != nil {
    fmt.Println(err)
    fmt.Println("Waiting 2 second timeout...")
    time.Sleep(time.Second * 2)
    continue
}

myImage = doWorkOnImage(myImage)
```

We now of course have to save the image back to the storage:

```go
myImage = doWorkOnImage(myImage)

err = sendImageToStorage(storageLocation, myTask, myImage)
if err != nil {
    fmt.Println(err)
    fmt.Println("Waiting 2 second timeout...")
    time.Sleep(time.Second * 2)
    continue
}
```

And finally if that succeeds we register our successfully finished _Task_!

```go
        err = sendImageToStorage(storageLocation, myTask, myImage)
        if err != nil {
            fmt.Println(err)
            fmt.Println("Waiting 2 second timeout...")
            time.Sleep(time.Second * 2)
            continue
        }

        err = registerFinishedTask(masterLocation, myTask)
        if err != nil {
            fmt.Println(err)
            fmt.Println("Waiting 2 second timeout...")
            time.Sleep(time.Second * 2)
            continue
        }
    }
}()
```

Ok, so now we can continue with implementing these functions. Let&#8217;s first implement the _getNewTask_ function:

```go
func getNewTask(masterAddress string) (Task, error) {
    response, err := http.Post("http://" + masterAddress + "/getNewTask", "text/plain", nil)
    if err != nil || response.StatusCode != http.StatusOK {
        return Task{-1, -1}, err
    }
    data, err := ioutil.ReadAll(response.Body)
    if err != nil {
        return Task{-1, -1}, err
    }

    myTask := Task{}
    err = json.Unmarshal(data, &myTask)
    if err != nil {
        return Task{-1, -1}, err
    }

    return myTask, nil
}
```

We make the request to the master and check if it was successful. We read the response _body_ to memory and finally _Unmarshal_ the response _body_ to our _Task_ structure. Finally we return it.

Now we can implement the function to get the image from storage!

```go
func getImageFromStorage(storageAddress string, myTask Task) (image.Image, error) {
    response, err := http.Get("http://" + storageAddress + "/getImage?state=working&id=" + strconv.Itoa(myTask.Id))
    if err != nil || response.StatusCode != http.StatusOK {
        return nil, err
    }

    myImage, err := png.Decode(response.Body)
    if err != nil {
        return nil, err
    }

    return myImage, nil
}
```

We get the response whose body is the raw image, so we just _Decode_ it and return it if we succeed.

Now that we have our image we can finally do some work on it:

```go
func doWorkOnImage(myImage image.Image) image.Image {
    myCanvas := image.NewRGBA(myImage.Bounds())

    for i := 0; i < myCanvas.Rect.Max.X; i++ {
        for j := 0; j < myCanvas.Rect.Max.Y; j++ {
            r, g, b, _ := myImage.At(i, j).RGBA()
            myColor := new(color.RGBA)
            myColor.R = uint8(g)
            myColor.G = uint8(r)
            myColor.B = uint8(b)
            myColor.A = uint8(255)
            myCanvas.Set(i, j, myColor)
        }
    }

    return myCanvas.SubImage(myImage.Bounds())
}
```

The work is largely irrelevant, but I&#8217;ll explain it anyways. First we create a RGBA. That&#8217;s something like a canvas for drawing, and we create it with the size of our image. Later we draw on the canvas swapping the red with the green channel. Later we use the RGBA to return a new modified image, created from our canvas with the size of our original image.

After working on the image we have to send it back to the _storage system_. So let&#8217;s implement the **_sendImageToStorage_** function:

```go
func sendImageToStorage(storageAddress string, myTask Task, myImage image.Image) error {
    data := []byte{}
    buffer := bytes.NewBuffer(data)
    err := png.Encode(buffer, myImage)
    if err != nil {
        return err
    }
    response, err := http.Post("http://" + storageAddress + "/sendImage?state=finished&id=" + strconv.Itoa(myTask.Id), "image/png", buffer)
    if err != nil || response.StatusCode != http.StatusOK {
        return err
    }

    return nil
}
```

We create a **data** _byte slice_, and from that a data _buffer_ which allows us to use it as a _readwriter_ interface. We then use this interface to encode our image to png into, and finally send it using a **POST** to the server. If everything works out, then we just return.

When we successfully saved the image, we can register to the _Master_ that we finished the _Task_.

```go
func registerFinishedTask(masterAddress string, myTask Task) error {
    response, err := http.Post("http://" + masterAddress + "/registerTaskFinished?id=" + strconv.Itoa(myTask.Id), "test/plain", nil)
    if err != nil || response.StatusCode != http.StatusOK {
        return err
    }

    return nil
}
```

Nothing fancy. Just sending a **POST** request to notify about finishing the task.

Ok, that&#8217;s all regarding the _worker_. You can turn it on and it will wait for _Tasks_ to get. Which means we need to get the _Tasks_ from the user to our service finally. And that leads us to&#8230;

## The Frontend

This one will show the user the website and also parse the user form so the backend services only get the raw image data. It will be the only interface to our application for the user.

As usual, we&#8217;ll begin with the basic structure of the file:

```go
package main

import (
    "net/http"
    "fmt"
    "io/ioutil"
    "os"
    "net/url"
    "io"
)

const indexPage = "<html><head><title>Upload file</title></head><body><form enctype=\"multipart/form-data\" action=\"submitTask\" method=\"post\"> <input type=\"file\" name=\"uploadfile\" /> <input type=\"submit\" value=\"upload\" /> </form> </body> </html>"

var keyValueStoreAddress string
var masterLocation string

func main() {
    if len(os.Args) < 2 {
        fmt.Println("Error: Too few arguments.")
        return
    }
    keyValueStoreAddress = os.Args[1]

    response, err := http.Get("http://" + keyValueStoreAddress + "/get?key=masterAddress")
    if response.StatusCode != http.StatusOK {
        fmt.Println("Error: can't get master address.")
        fmt.Println(response.Body)
        return
    }
    data, err := ioutil.ReadAll(response.Body)
    if err != nil {
        fmt.Println(err)
        return
    }
    masterLocation = string(data)
    if len(masterLocation) == 0 {
        fmt.Println("Error: can't get master address. Length is zero.")
        return
    }

    http.HandleFunc("/", handleIndex)
    http.HandleFunc("/submitTask", handleTask)
    http.HandleFunc("/isReady", handleCheckForReadiness)
    http.HandleFunc("/getImage", serveImage)
    http.ListenAndServe(":80", nil)
}

func handleIndex(w http.ResponseWriter, r *http.Request) {
}

func handleTask(w http.ResponseWriter, r *http.Request) {
}

func handleCheckForReadiness(w http.ResponseWriter, r *http.Request) {
}

func serveImage(w http.ResponseWriter, r *http.Request) {
}
```

We&#8217;ve got the code of our index web page here, and we also have the **_API_** declared. After starting the program we check if we have the k/v store address. If we do (we hope so), then we can get the _master address_ from it.

Now we can go on to implementing the functions. We&#8217;ll start with the simples. The index handler:

```go
func handleIndex(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, indexPage)
}
```

After writing this we can go on to writing the more complicated functions. We will start with the _handleTask_ function. This function is responsible for parsing the user form and sending the raw image data to the master.

```go
func handleTask(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodPost {
    err := r.ParseMultipartForm(10000000)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Wrong input")
        return
    }
    file, _, err := r.FormFile("uploadfile")
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Wrong input")
        return
    }
```

Ok, what do we have here? We check the method as we always do, and later parse the multipart form. We&#8217;ve got a nice lovely magic number there. The number is responsible for setting the max size of the form held in RAM. The rest will be stored in temporary files. We later do the request using the file we got:

```go
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Wrong input")
        return
    }

    response, err := http.Post("http://" + masterLocation + "/new", "image", file)
    if err != nil || response.StatusCode != http.StatusOK {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error:", err)
        return
    }

    data, err := ioutil.ReadAll(response.Body)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error:", err)
        return
    }

    fmt.Fprint(w, string(data))
} else {
    w.WriteHeader(http.StatusBadRequest)
    fmt.Fprint(w, "Error: Only POST accepted")
}
```

We send the user back the new _Task id_ we got back from the _master_.

Now we can implement the function which will handle the request to check if the _Task_ is finished and ready:

```go
func handleCheckForReadiness(w http.ResponseWriter, r *http.Request) {
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

        response, err := http.Get("http://" + masterLocation + "/isReady?id=" + values.Get("id") + "&state=finished")
        if err != nil || response.StatusCode != http.StatusOK {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", err)
            return
        }

        data, err := ioutil.ReadAll(response.Body)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", err)
            return
        }

        switch string(data) {
        case "0":
            fmt.Fprint(w, "Your image is not ready yet.")
        case "1":
            fmt.Fprint(w, "Your image is ready.")
        default :
            fmt.Fprint(w, "Internal server error.")
        }
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only GET accepted")
    }
}
```

Most of this is the same as the function in the _master_. We just check if the id is correct and make a request to the _master_. The interesting part is this:

```go
switch string(data) {
case "0":
    fmt.Fprint(w, "Your image is not ready yet.")
case "1":
    fmt.Fprint(w, "Your image is ready.")
default :
    fmt.Fprint(w, "Internal server error.")
}
```

We cast the _response body_ to a string, and if it matches 1 or 0 we answer. If it&#8217;s something else then we know something went really wrong, and send back an error.

Now we can get to the last function, the function to serve the actual images:

```go
func serveImage(w http.ResponseWriter, r *http.Request) {
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

        response, err := http.Get("http://" + masterLocation + "/get?id=" + values.Get("id") + "&state=finished")
        if err != nil || response.StatusCode != http.StatusOK {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", err)
            return
        }

        _, err = io.Copy(w, response.Body)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprint(w, "Error:", err)
            return
        }
    } else {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "Error: Only GET accepted")
    }
}
```

It just checks the _id_, sends the _request_, and copies the _response_ to answer the user.

## Conclusion

So now it&#8217;s all finished. You can start all applications and they will work. You can contact the frontend and make requests, they will all start working and you will get your modified images. Six microservices working beautifully together.

This may be the end of this series, so have fun extending this system alone.

I may add a finishing part about deployment to container infrastructues, or a another extending series about refactoring the system with 3rd party libraries in the future but I&#8217;m not sure.

All in all, good luck!

UPDATE: Also remember, that when running the worker, it&#8217;s good to launch it with goroutines in the number of 2-4x your system threads. They have near-0 overhead in switching and this way your not unnecessarily blocking when waiting for http responses.

 [1]: https://jacobmartins.com/2016/03/21/web-app-using-microservices-in-go-part-3-storage-and-master/