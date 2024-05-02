---
title: 'Practical Golang: Using Google Drive and Calendar'
author: Jacob Martin
type: legacy-posts
date: 2016-03-08T13:47:01+00:00
url: /2016/03/08/practical-golang-using-google-drive-and-calendar/
categories:
  - Go
  - Practical Golang

---
## Introduction

Integrating Google services into your app can lead to a lot of nice features for your users, and can create a seamless experience for them. In this tutorial we&#8217;ll learn how to use the most useful functionalities of **_Google Calendar_** and **_Google Drive_**.

## The theory

To begin with, we should understand the methodology of using the **_Google API_** in Golang. For most of their API&#8217;s I&#8217;ve skimmed through it works like that:

  1. Create an **OAuth2 client** from the OAuth2 _access token_.
  2. Use the client to create an app service, this will be our interface we&#8217;ll use to communicate with Google services.
  3. We create a request object and set the needed parameters.
  4. We start the action, usually using the _Do()_ function on the request object.

After you learn it for one Google service, it will be trivial for you to use it for any other.

## Dependencies

Here we will need the Google Calendar and Google Drive libraries, both of which we need version 3 of. (the newest at the moment)

So make sure to:

    go get google.golang.org/api/calendar/v3
    go get google.golang.org/api/drive/v3
    

## The basic structure

For both of our apps we&#8217;ll need the same basic OAuth2 app structure. You can learn more about it in [my previous article][1]

```go
package main

import (
    "fmt"
    "net/http"
    "golang.org/x/oauth2"
    "os"
    "golang.org/x/oauth2/google"
    "golang.org/x/net/context"
    "time"
)

var (
    googleOauthConfig = &oauth2.Config{
        RedirectURL:    "http://localhost:3000/GoogleCallback",
        ClientID:     os.Getenv("googlekey"), // from https://console.developers.google.com/project/<your-project-id>/apiui/credential
        ClientSecret: os.Getenv("googlesecret"), // from https://console.developers.google.com/project/<your-project-id>/apiui/credential
        Scopes:       []string{},
        Endpoint:     google.Endpoint,
    }
// Some random string, random for each request
    oauthStateString = "random"
)

const htmlIndex = `<html><body>
<a href="/GoogleLogin">Log in with Google</a>
</body></html>
`

func main() {
    http.HandleFunc("/", handleMain)
    http.HandleFunc("/GoogleLogin", handleGoogleLogin)
    http.HandleFunc("/GoogleCallback", handleGoogleCallback)
    fmt.Println(http.ListenAndServe(":3000", nil))
}
func handleMain(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, htmlIndex)
}

func handleGoogleLogin(w http.ResponseWriter, r *http.Request) {
    url := googleOauthConfig.AuthCodeURL(oauthStateString)
    http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}

func handleGoogleCallback(w http.ResponseWriter, r *http.Request) {
    state := r.FormValue("state")
    if state != oauthStateString {
        fmt.Printf("invalid oauth state, expected '%s', got '%s'\n", oauthStateString, state)
        http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
        return
    }

    code := r.FormValue("code")
    token, err := googleOauthConfig.Exchange(oauth2.NoContext, code)
    if err != nil {
        fmt.Printf("oauthConf.Exchange() failed with '%s'\n", err)
        http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
        return
    }

    client := oauth2.NewClient(context.Background(), oauth2.StaticTokenSource(token))
}
```

There is one thing I haven&#8217;t covered in my previous blog post, namely the last line:

```go
client := oauth2.NewClient(context.Background(), oauth2.StaticTokenSource(token))
```

We need an **_OAuth2 client_** to use the Google API, so we create one. It takes a _context_, for lack of which we just use the _background context_. It also needs a _token source_. As we only want to make one request and know that this token will suffice we create a _static token source_ which will always generate the same _token_ which we&#8217;ve passed to it.

## Creating the Calendar app.

### Preperation

First, as described in [my previous article][1], you should enable the Google Calendar API in the **_Google Cloud Console_** for you app.

Also, we&#8217;ll need to ask for permission, so add the **_https://www.googleapis.com/auth/calendar_** scope to our googleOauthConfig:

```go
googleOauthConfig = &oauth2.Config{
        RedirectURL:    "http://localhost:3000/GoogleCallback",
        ClientID:     os.Getenv("googlekey"), // from https://console.developers.google.com/project/<your-project-id>/apiui/credential
        ClientSecret: os.Getenv("googlesecret"), // from https://console.developers.google.com/project/<your-project-id>/apiui/credential
        Scopes:       []string{"https://www.googleapis.com/auth/calendar"},
        Endpoint:     google.Endpoint,
    }
```

Remember to import google.golang.org/api/calendar/v3

### The main code

We&#8217;ll add everything we write right after creating our OAuth2 client.

First, as I described before, we&#8217;ll need an app service, here it will be the calendar service, so let&#8217;s create it!

```go
client := oauth2.NewClient(context.Background(), oauth2.StaticTokenSource(token))

calendarService, err := calendar.New(client)
if err != nil {
  fmt.Fprintln(w, err)
  return
}
```

It just uses the OAuth client to create the service and errors out if something goes wrong.

#### Listing events

Now we will create a request, add a few optional parameters to it and start it. We&#8217;ll build it up step by step.

```go
calendarService, err := calendar.New(client)
if err != nil {
  fmt.Fprintln(w, err)
  return
}

calendarService.Events.List("primary")
```

This creates a request to list all events in your primary calendar. You could also name a specific calendar, but using _primary_ will take the primary calendar of that user.

So&#8230; I think we don&#8217;t really care about the events 5 years ago. So let&#8217;s only take the upcoming ones.

```go
calendarService.Events.List("primary").TimeMin(time.Now().Format(time.RFC3339))
```

We add an option _TimeMin_ which takes a _DateTime_ by string&#8230; No idea why it isn&#8217;t a nice struct like _time.DateTime_. You also need to format it as a string in the RFC3339 format.

Ok&#8230; but that could be a lot of events, so we&#8217;ll just take the 5 first:

```go
calendarService.Events.List("primary").TimeMin(time.Now().Format(time.RFC3339)).MaxResults(5)
```

Now we just have to **_Do()_** it, and store the results:

```go
calendarEvents, err := calendarService.Events.List("primary").TimeMin(time.Now().Format(time.RFC3339)).MaxResults(5).Do()
if err != nil {
  fmt.Fprintln(w, err)
  return
}
```

How can we now do something with the results? Simple! :

```go
calendarEvents, err := calendarService.Events.List("primary").TimeMin(time.Now().Format(time.RFC3339)).MaxResults(5).Do()
if err != nil {
  fmt.Fprintln(w, err)
  return
}

if len(calendarEvents.Items) > 0 {
    for _, i := range calendarEvents.Items {
        fmt.Fprintln(w, i.Summary, " ", i.Start.DateTime)
    }
}
```

We access a list of events using the **_Items_** field in the _calendarEvents_ variable, if there is at least one element, then for each element we write the _summary_ and _start time_ to the _response writer_ using a _for range_ loop.

#### Creating an event

Ok, we already know how to list events, now let&#8217;s create an event!  
First, we need an event object:

```go
if len(calendarEvents.Items) > 0 {
    for _, i := range calendarEvents.Items {
        fmt.Fprintln(w, i.Summary, " ", i.Start.DateTime)
    }
}
newEvent := calendar.Event{
    Summary: "Testevent",
    Start: &calendar.EventDateTime{DateTime: time.Date(2016, 3, 11, 12, 24, 0, 0, time.UTC).Format(time.RFC3339)},
    End: &calendar.EventDateTime{DateTime: time.Date(2016, 3, 11, 13, 24, 0, 0, time.UTC).Format(time.RFC3339)},
}
```

We create an Event struct and pass in the **_summary_** &#8211; title of the event.  
We also pass the start and finish **_DateTime_**. We create a _date_ using the stdlib _time_ package, and then convert it to a string in the RFC3339 format. There are tons of other optional fields you can specify if you want to.

Next we need to create an **_insert_** request object:

```go
newEvent := calendar.Event{
    Summary: "Testevent",
    Start: &calendar.EventDateTime{DateTime: time.Date(2016, 3, 11, 12, 24, 0, 0, time.UTC).Format(time.RFC3339)},
    End: &calendar.EventDateTime{DateTime: time.Date(2016, 3, 11, 13, 24, 0, 0, time.UTC).Format(time.RFC3339)},
}
calendarService.Events.Insert("primary", &newEvent)
```

The **_Insert_** request takes two arguments, the calendar name and an event object.

As usual we neeed to **_Do()_** the request! and saving the resulting created event can also come handy in the future:

```go
createdEvent, err := calendarService.Events.Insert("primary", &newEvent).Do()
if err != nil {
    fmt.Fprintln(w, err)
    return
}
```

In the end let&#8217;s just print some kind of confirmation to the user:

```go
fmt.Fprintln(w, "New event in your calendar: \"", createdEvent.Summary, "\" starting at ", createdEvent.Start.DateTime)
```

### Hint

You can define the event **_ID_** yourself before creating it, but you can also let the _Google Calendar_ service generate an **_ID_** for us as we did.

## Creating the Drive app

### Preperation

First, enable the Google Drive API in the **_Google Cloud Console_**

Again we will need to ask the user for permission. So we have to use the https://www.googleapis.com/auth/drive and https://www.googleapis.com/auth/drive.file scopes:

```go
googleOauthConfig = &oauth2.Config{
        RedirectURL:    "http://localhost:3000/GoogleCallback",
        ClientID:     os.Getenv("googlekey"), // from https://console.developers.google.com/project/<your-project-id>/apiui/credential
        ClientSecret: os.Getenv("googlesecret"), // from https://console.developers.google.com/project/<your-project-id>/apiui/credential
        Scopes:       []string{"https://www.googleapis.com/auth/drive", "https://www.googleapis.com/auth/drive.file"},
        Endpoint:     google.Endpoint,
    }
```

Remember to import google.golang.org/api/drive/v3&#8243;

### The main code

Again we need to start from our basic OAuth2 app structure with an OAuth2 client, and create an app service. Here, this will be the **_Drive service_**:

```go
client := oauth2.NewClient(context.Background(), oauth2.StaticTokenSource(token))

driveService, err := drive.New(client)
if err != nil {
    fmt.Fprintln(w, err)
    return
}
```

#### Listing files

Ok, to begin with let&#8217;s learn how to list files from the **_Google Drive_**. There is an important concept you should understand before we start. Whenever we request a list of files from _Google Drive_, those can literally be thousands of files. That&#8217;s why we get one page of files, which includes files metadata, not the content, and a **_NextPageToken_**, which we can use to get the next page.

As before with the calendar app, we create a list request.

```go
driveService, err := drive.New(client)
if err != nil {
    fmt.Fprintln(w, err)
    return
}

driveService.Files.List()
```

But we can also set an option. The ordering for example:

```go
driveService.Files.List().OrderBy("name")
```

Ok, now let&#8217;s **_Do()_** it and save the results:

```go
myFilesList, err := driveService.Files.List().OrderBy("name").Do()
if err != nil {
    fmt.Fprintf(w, "Couldn't retrieve files ", err)
}
```

As I wrote before, this is one page of files, now let&#8217;s go over it:

```go
if len(myFilesList.Files) > 0 {
    for _, i := range myFilesList.Files {
        fmt.Fprintln(w, i.Name, " ", i.Id)
    }
} else {
    fmt.Fprintln(w, "No files found.")
}
```

We check if there are any files, and if there are, we range over them printing the _name_ and _Id_ of each one, else we print that there are none.

Now, let&#8217;s get more pages, it&#8217;s the **_myFilesList.NextPageToken_** holding the token for the next page. If it is an empty string, then this was the last page. While it is present we load new pages into our _myFilesList_ variable.

```go
for myFilesList.NextPageToken != "" {
    myFilesList, err = driveService.Files.List().OrderBy("name").PageToken(myFilesList.NextPageToken).Do()
    if err != nil {
        fmt.Fprintf(w, "Couldn't retrieve files ", err)
        break
    }
    fmt.Fprintln(w, "Next Page: !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
    if len(myFilesList.Files) > 0 {
        for _, i := range myFilesList.Files {
            fmt.Fprintln(w, i.Name, " ", i.Id)
        }
    } else {
        fmt.Fprintln(w, "No files found.")
    }
}
```

To retrieve the next page we add the **_PageToken_** option to our file listing request

```go
    myFilesList, err = driveService.Files.List().OrderBy("name").PageToken(myFilesList.NextPageToken).Do()
```

Whenever we start printing the new page, we notify a user that a new page just started. Later we check if we have any files, and if we do, then we range over them as before, printing the _names_ and _Id&#8217;s_

#### Creating a file

We already know how to list files in our _Google Drive_. Now let&#8217;s learn how to create a file and add some content to it!

First, we need to create the file metadata:

```go
for myFilesList.NextPageToken != "" {
    myFilesList, err = driveService.Files.List().OrderBy("name").PageToken(myFilesList.NextPageToken).Do()
    if err != nil {
        fmt.Fprintf(w, "Couldn't retrieve files ", err)
        break
    }
    fmt.Fprintln(w, "Next Page: !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
    if len(myFilesList.Files) > 0 {
        for _, i := range myFilesList.Files {
            fmt.Fprintln(w, i.Name, " ", i.Id)
        }
    } else {
        fmt.Fprintln(w, "No files found.")
    }
}


myFile := drive.File{Name: "cats.png"}
```

Again, the _Id_ will be generated for us if we do not provide any.

Putting the file into _Google Drive_ is pretty easy now. We create a _create request_, referencing our file metadata in it:

```go
myFile := drive.File{Name: "cats.png"}

driveService.Files.Create(&myFile)
```

and **_Do()_** it, saving the results:

```go
createdFile, err := driveService.Files.Create(&myFile).Do()
if err != nil {
    fmt.Fprintf(w, "Couldn't create file ", err)
}
```

Ok, if you started this code, you would get a **_cats.png_** file. However, it&#8217;s empty. So let&#8217;s add some data to it!

To add it, we&#8217;ll first need some data. We can load it from a file:

```go
createdFile, err := driveService.Files.Create(&myFile).Do()
if err != nil {
    fmt.Fprintf(w, "Couldn't create file ", err)
}
myImage, err := os.Open("/tmp/image.png")
if err != nil {
    fmt.Fprintln(w, err)
}
```

Now we have to create the updated file metadata:

```go
myImage, err := os.Open("/tmp/image.png")
if err != nil {
    fmt.Fprintln(w, err)
}
updatedFile := drive.File{Name: "catsUpdated.png"}
```

We have to construct an update request:

```go
driveService.Files.Update(createdFile.Id, &updatedFile)
```

Here we specify the **_Id_** of the file to modify, and the new metadata. We&#8217;ll now add the data to the update request, and **_Do()_** it, checking for errors and ignoring the result.

```go
_, err = driveService.Files.Update(createdFile.Id, &updatedFile).Media(myImage).Do()
if err != nil {
    fmt.Fprintln(w, err)
}
fmt.Fprintln(w, createdFile.Id)
```

In the end we send the new file id to the user.

#### Hint

You could add the **_Media_** option already to the create request if you wanted.

## Conclusion

I suppose this was a good short introduction into the structure of the **_Google API_**. After learning these, you should have few to no problems using the other API&#8217;s.

Have fun integrating it into your app!

 [1]: https://kubamartin.com/2016/02/29/getting-started-with-oauth2-in-go/