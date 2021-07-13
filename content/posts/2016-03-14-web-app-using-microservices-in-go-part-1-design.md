---
title: 'Web app using Microservices in Go: Part 1 – Design'
author: Jacob Martin
type: legacy-posts
date: 2016-03-14T14:19:19+00:00
url: /2016/03/14/web-app-using-microservices-in-go-part-1-design/
categories:
  - Go
tags:
  - architecture
  - design
  - go
  - golang
  - microservice
  - microservices
  - web app

---
## Introduction

Recently it&#8217;s a constantly repeated buzzword &#8211; **_Microservices_**. _You can love &#8217;em or hate &#8217;em, but you really shouldn&#8217;t ignore &#8217;em_. In this short series we&#8217;ll create a web app using a microservice architecture. We&#8217;ll try not to use 3rd party tools and libraries. Remember though that when creating a production web app it is highly recommended to use 3rd party libraries (even if only to save you time).

We will create the various components in a basic form. We won&#8217;t use advanced _caching_ or use a _database_. We will create a basic **key-value** store and a simple storage service. We will use the **_Go_** language for all this.

<strike>UPDATE: as there are comments regarding overcomplication: this is meant to show a scalable and working skeleton for a microservice architecture. If you only want to add some filters to photos, don&#8217;t design it like that. It&#8217;s overkill.</strike>

On further thought and another comment, (Which you can find on the golang Reddit) do design it this way. Software usually lives much longer than we think it will, and such a design will lead to an easily extendable and scalable web app.

## The functionality

First we should decide what our web app will do. The web app we&#8217;ll create in this series will get an image from a user and give back an unique **ID**. The image will get modified using complicated and highly sophisticated algorithms, like swapping the blue and red channel, and the user will be able to use the _ID_ to check if the work on the image has been finished already or if it&#8217;s still in progress. If it&#8217;s finished he will be able to download the altered image.

## Designing the architecture

We want the **architecture** to be microservices, so we should design it like that. We&#8217;ll for sure need a service facing the user, the one that provides the _interface_ for communication with our app. This could also handle _authentication_, and should be used as the service redirecting the workload to the right sub-services. (useful if you plan to integrate more funcionality into the app)

We will also want a microservice which will handle all our images. It will get the image, generate an _ID_, store information related to each _task_, and save the images. To handle high workloads it&#8217;s a good idea to use a **_master-slave_** system for our image modification service. The image handler will be the _master_, and we will create _slave_ microservices which will ask the _master_ for images to work on.

We will also need a _key-value_ datastore for various _configuration_, a storage system, for saving our images, pre- and post-modification, and a database-ish service holding the information about each task.

This should suffice to begin with.

Here I&#8217;d like to also state that the architecture could change during the series if needed. And I encourage you to comment if you think that something could be done better.

### Communication

We will also need to define the method the services communicate by. In this app we will use **_REST_** everywhere. You could also use a **_message BUS_** or **_Remote Procedure Calls_** &#8211; short **_RPC_**, but I won&#8217;t write about them here.

### Designing the microservice API&#8217;s

Another important thing is to design the **_API_**&#8216;s of you microservices. We will now design each of them to get an understanding about what they are for.

#### The key-value store

This one&#8217;s mainly for configuration. It will have a simple post-get interface:

  * POST: 
      * Arguments: 
          * Key
          * Value
      * Response: 
          * Success/Failure
  * GET: 
      * Arguments: 
          * Key
      * Response: 
          * Value/Failure

#### The storage

Here we will store the images, again using a key-value interface and an argument stating if this one&#8217;s pre- or post-modification. For the sake of simplicity we will just save the image to a folder named, depending on the state of the image, finished/inProgress.

  * POST: 
      * Arguments: 
          * Key
          * State: pre-/post-modification
          * Data
      * Response: 
          * Success/Failure
  * GET: 
      * Arguments: 
          * Key
          * State: pre-/post-modification
      * Response: 
          * Data/Failure

#### Database

This one will save our tasks. If they are waiting to start, in progress or finished, their Id.

  * POST: 
      * Arguments: 
          * TaskId
          * State: not started/ in progress/ finished
      * Response: 
          * Success/Failure
  * GET: 
      * Arguments: 
          * TaskId
      * Response: 
          * State/Failure
  * GET: 
      * Path: 
          * not started/ in progress/ finished
      * Reponse: 
          * list of TaskId&#8217;s

#### The Frontend

The frontend is there mainly to provide a communication way between the various services and the user. It can also be used for authentication and authorization.

  * POST: 
      * Path: 
          * newImage
      * Arguments: 
          * Data
      * Response: 
          * Id
  * GET: 
      * Path: 
          * image/isReady
      * Arguments: 
          * Id
      * Response: 
          * not found/ in progress / finished
  * GET: 
      * Path: 
          * image/get
      * Arguments: 
          * Id
      * Response: 
          * Data

#### Image master microservice

This one will get new images from the fronted/user and send them to the storage service. It will also create a new task in the database, and orchestrate the workers who can ask for work and notify when it&#8217;s finished.

  * Frontend interface: 
      * POST: 
          * Path: 
              * newImage
          * Arguments: 
              * Data
          * Response: 
              * Id
      * GET: 
          * Path: 
              * isReady
          * Arguments: 
              * Id
          * Response: 
              * not found/ in progress / finished
      * GET: 
          * Path: 
              * get
          * Arguments: 
              * Id
          * Response: 
              * Data/Failure
  * Worker interface: 
      * GET: 
          * Path: 
              * getWork
          * Response: 
              * Id/noWorkToDo
      * POST: 
          * Path: 
              * workFinished
          * Arguments: 
              * Id
          * Response: 
              * Success/Failure

#### Image worker microservice

This one doesn&#8217;t have any API. It is a client to the master image service, which he finds using the key-value store. He gets the image data to work on from the storage service.

## Scheme

![The microservice architecure design][1] 

## Conclusion

This is basically everything regarding the design. In the next part we will write part of the microservices. Again, I encourage you to comment expressing what you think about this design!

[Next part][2]

 [1]: https://www.lucidchart.com/publicSegments/view/cb49c63f-9256-47ae-a21a-18afa85cc4fd/image.png
 [2]: https://jacobmartins.com/2016/03/16/web-app-using-microservices-in-go-part-2-kv-store-and-database/