---
title: Getting started with OAuth2 in Go
author: Jacob Martin
type: legacy-posts
date: 2016-02-29T14:02:02+00:00
url: /2016/02/29/getting-started-with-oauth2-in-go/
categories:
  - Go
tags:
  - beginner
  - go
  - golang
  - oauth2
  - web app
---
## Introduction

Authentication is usually a crucial part in any web app. You could always roll your own authentication mechanics if you wanted, however, this creates an additional barrier between the user and your web app: Registration.

That&#8217;s why OAuth, and earlier OAuth2, was created. It makes it much more convenient to log in to your app, because the user can log in with one of the many accounts he already has.

## What we&#8217;ll cover in this tutorial

We will set up a web app with OAuth2 provided by Google. For this we&#8217;ll need to:  
1. Create a web app in Google and get our ClientID and a ClientSecret.  
2. Put those into accessible and fairly safe places in our system.  
3. Plan the structure of our web app.  
4. Make sure we have the needed dependencies.  
5. Understand how OAuth2 works.  
6. Write the application logic.

**Let&#8217;s begin.**

## Creating a project in Google and getting the client ID and secret

First, go to the [Google Cloud Platform][1] and create a new project. Later open the left menu, and open the **_API Manager_**. There, search for the **_Google+ API_** and enable it.

Next, open the credentials submenu, and choose **_Create credentials -> OAuth client ID_**. The application type should be **_Web application_** and give it a custom name if you want. In &#8220;Authorized JavaScript origins&#8221; put in the address of the site you&#8217;ll be login in from. I will use http://localhost:3000. Then, in the field **_Authorized redirect URLs_** put in the address of the site, to which the user will be redirected after logging in. I&#8217;ll use http://localhost:3000/GoogleCallback.

Now the **_client ID_** and **_client secret_** should be displayed for you. Write them down somewhere safe. Remember that the client secret has to stay secret for the entire lifetime of your app.

## Safely storing the client ID and secret

There are many ways to safely store the client ID and secret. In production you should make sure that the client secret remains secret.

In this tutorial we won&#8217;t cover this. Instead, we will store those variables as system environment variables. Now:  
* Create an environment variable called **_googlekey_** holding your client ID.  
* Create an environment variable called **_googlesecret_** holding your client secret.

## Planning the structure

In this tutorial we&#8217;ll write code in one file. In production you would want to split this into multiple files.

Let&#8217;s start with a basic go web app structure:

```go
package main

import (
  "fmt"
  "net/http"
)

func main() {
  fmt.Println(http.ListenAndServe(":3000", nil))
}
```

Now we&#8217;ll set up a simple site:

```go
const htmlIndex = `&lt;html&gt;&lt;body&gt;
&lt;a href="/GoogleLogin"&gt;Log in with Google&lt;/a&gt;
&lt;/body&gt;&lt;/html&gt;
`
```

We will also need:  
* The home page, where we will click the login button from.  
* The page handling redirection to the google service.  
* The callback page handling the information we get from Google.

So let&#8217;s set up the base structure for that:

```go
func main() {
    http.HandleFunc("/", handleMain)
    http.HandleFunc("/GoogleLogin", handleGoogleLogin)
    http.HandleFunc("/GoogleCallback", handleGoogleCallback)
    fmt.Println(http.ListenAndServe(":3000", nil))
}

func handleMain(w http.ResponseWriter, r *http.Request) {
}

func handleGoogleLogin(w http.ResponseWriter, r *http.Request) {
}

func handleGoogleCallback(w http.ResponseWriter, r *http.Request) {
}
```

## Dependencies

You will need to

    go get golang.org/x/oauth2
    

if you don&#8217;t have it already.

## Understanding OAuth2

To really integrate OAuth2 into our web application it&#8217;s good to understand how it works.  
That&#8217;s the flow of OAuth2:  
1. The user opens the website and clicks the login button.  
2. The user gets redirected to the google login handler page. This page generates a random state string by which it will identify the user, and constructs a google login link using it. The user then gets redirected to that page.  
3. The user logs in and gets back a code and the random string we gave him. He gets redirected back to our page, using a POST request to give us the code and state string.  
4. We verify if it&#8217;s the same state string. If it is then we use the code to ask google for a short-lived **_access token_**. We can save the code for future use to get another token later.  
5. We use the **_token_** to initiate actions regarding the user account.

## Writing the application logic

Before starting remember to import the _golang.org/x/oauth2_ package.  
To begin with, let&#8217;s write the home page handler:

```go
func handleMain(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, htmlIndex)
}
```

Next we need to create a variable we&#8217;ll use for storing data and communicating with Google and the **_random state variable_**:

```go
var (
    googleOauthConfig = &oauth2.Config{
        RedirectURL:    "http://localhost:3000/GoogleCallback",
        ClientID:     os.Getenv("googlekey"),
        ClientSecret: os.Getenv("googlesecret"),
        Scopes:       []string{"https://www.googleapis.com/auth/userinfo.profile",
            "https://www.googleapis.com/auth/userinfo.email"},
        Endpoint:     google.Endpoint,
    }
// Some random string, random for each request
    oauthStateString = "random"
)
```

The _Scopes_ variable defines the amount of access we get over the users account.

Note that the _oauthStateString_ should be randomly generated on a per user basis.

#### Handling communication with Google

This is the code that creates a login link and redirects the user to it:

```go
func handleGoogleLogin(w http.ResponseWriter, r *http.Request) {
    url := googleOauthConfig.AuthCodeURL(oauthStateString)
    http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}
```

We use the _googleOauthConfig_ variable to create a login link using the random state variable, and later redirect the user to it.

* * *

Now we need the logic that get&#8217;s the code after the user logs in and checks if the state variable matches:

```go
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
        fmt.Println("Code exchange failed with '%s'\n", err)
        http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
        return
    }

    response, err := http.Get("https://www.googleapis.com/oauth2/v2/userinfo?access_token=" + token.AccessToken)

    defer response.Body.Close()
    contents, err := ioutil.ReadAll(response.Body)
    fmt.Fprintf(w, "Content: %s\n", contents)
}
```

First we check the state variable, and notify the user if it doesn&#8217;t match. If it matches we get the code and communicate with google using the _Exchange_ function. We have no context so we use _NoContext_.

Later, if we successfully get the token we make a request to google passing the token with it and get the users _userinfo_. We print the response to our user.

## Conclusion

That&#8217;s all we have to do to integrate OAuth2 into our Golang application. I hope that I helped someone with this problems as I really couldn&#8217;t find beginner-suited, detailed resources about OAuth2 in Go.

_Now go and build something amazing!_

**Update:** if you plan to integrate OAuth2 into the Authentication of your app, make sure to read this too: <http://oauth.net/articles/authentication/>  
Thanks to **_tbroyer_** for providing the link.

 [1]: https://console.developers.google.com