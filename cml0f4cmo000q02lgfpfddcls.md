---
title: "How to OAuth from the Command Line"
datePublished: Sun Nov 22 2020 10:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cml0f4cmo000q02lgfpfddcls
slug: how-to-oauth-from-the-command-line
canonical: https://dev.to/lauravuo/how-to-oauth-from-the-command-line-47j0
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1769748814270/ebfd7e4f-c74f-4d93-aa7f-b782f1f2eacd.webp
tags: go, oauth, command-line, spotify

---

This week I came up with an idea about a personal Christmas radio. I am an eager listener of Christmas tunes and every year I try to find some new favorite songs. I decided to code me a little helper that would ease the discovery process and automatically add new Christmas songs to my chosen Spotify playlist.

However, I encountered an authentication-related problem when implementing the functionality. [Accessing the Spotify user APIs](https://developer.spotify.com/documentation/general/guides/authorization-guide/) (such as modifying the playlists on behalf of the user) requires permission both from Spotify and the user. The user's permission is acquired through browser interaction (user logs in to Spotify and authorizes the app in the web UI), but my app was designed to work from the command line. So the problem was how to make the command line program interact with the browser flow?

The Spotify API authentication is implemented according to the popular OAuth 2.0 specification. I decided to use [the authorization code flow](https://tools.ietf.org/html/rfc6749#section-1.3.1) that would suit best for my purposes. The flow has two parts: first, the client application (my radio app) directs the user to an authorization server that handles the user authentication. Then the authorization server directs the request back to the client application (to a predefined redirect URI). With this redirect is delivered a code that the client application can use for requesting the actual API access token from another endpoint.

Although this protocol may sound a bit complex at first, it provides important security benefits: the user's credentials are never shared with the client application. And on the other hand, the final access token is delivered directly to the client application. This way it's not being exposed to the browser or the user and actually, only the client application can receive the token.

So the solution was to use a temporary localhost webserver. The server lives only as long as the redirect endpoint is called and the authorization code is received. The following figure describes the steps:

![Alt Text](https://cdn.hashnode.com/res/hashnode/image/upload/v1769748812593/c31fe2a1-7c87-4731-a423-13c1039b76dd.png)

1. The user launches the app from the command line
1. The client program starts the temporary server
1. The client program launches the browser to the API authentication page.
1. The user authenticates in the browser and authorizes the client application to access the API on her behalf.
1. The authorization server redirects the request to the predefined redirect URL (localhost).
1. The client program parses the redirect request and receives the authorization code. 
1. The client program exchanges the authorization code to the API access token calling the authorization server endpoint.
1. The client program receives the API access token and can make API requests on behalf of the user.

I used [golang](https://golang.org/) to implement the app and the sample code is attached here:

```go
package main

import (
	"context"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"net/url"
	"os"

	"github.com/pkg/browser"
)

type AuthResponse struct {
	AccessToken string `json:"access_token"`
}

func fetchUserToken() string {
	const (
		redirectURL     = "http://localhost:4321"
		spotifyLoginURL = "https://accounts.spotify.com/authorize?client_id=%s&response_type=code&redirect_uri=%s&scope=%s&state=%s"
	)

	var (
		clientID     = os.Getenv("SPOTIFY_CLIENT_ID")
		clientSecret = os.Getenv("SPOTIFY_CLIENT_SECRET")
		authHeader   = fmt.Sprintf("Basic %s", base64.StdEncoding.EncodeToString([]byte(clientID+":"+clientSecret)))
	)

	if clientID == "" && clientSecret == "" {
		panic(fmt.Errorf("spotify client ID and secret missing"))
	}

	// authorization code - received in callback
	code := ""
	// local state parameter for cross-site request forgery prevention
	state := fmt.Sprint(rand.Int())
	// scope of the access: we want to modify user's playlists
	scope := "playlist-modify-public&playlist-modify-private"
	// loginURL
	path := fmt.Sprintf(spotifyLoginURL, clientID, redirectURL, scope, state)

	// channel for signaling that server shutdown can be done
	messages := make(chan bool)

	// callback handler, redirect from authentication is handled here
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// check that the state parameter matches
		if s, ok := r.URL.Query()["state"]; ok && s[0] == state {
			// code is received as query parameter
			if codes, ok := r.URL.Query()["code"]; ok && len(codes) == 1 {
				// save code and signal shutdown
				code = codes[0]
				messages <- true
			}
		}
		// redirect user's browser to spotify home page
		http.Redirect(w, r, "https://www.spotify.com/", http.StatusSeeOther)
	})

	// open user's browser to login page
	if err := browser.OpenURL(path); err != nil {
		panic(fmt.Errorf("failed to open browser for authentication %s", err.Error()))
	}

	server := &http.Server{Addr: ":4321"}
	// go routine for shutting down the server
	go func() {
		okToClose := <-messages
		if okToClose {
			if err := server.Shutdown(context.Background()); err != nil {
				log.Println("Failed to shutdown server", err)
			}
		}
	}()
	// start listening for callback - we don't continue until server is shut down
	log.Println(server.ListenAndServe())

	// authentication complete - fetch the access token
	params := url.Values{}
	params.Add("grant_type", "authorization_code")
	params.Add("code", code)
	params.Add("redirect_uri", redirectURL)
	data, err := doPostRequest(
		"https://accounts.spotify.com/api/token",
		params,
		authHeader,
	)
	if err == nil {
		response := AuthResponse{}
		if err = json.Unmarshal(data, &response); err == nil {
			// happy end: token parsed successfully
			return response.AccessToken
		}
	}
	panic(fmt.Errorf("unable to acquire Spotify user token"))
}
```

Go provides quite nice tools for implementing this kind of concurrency handling that is needed here: the localhost server is shut down using [goroutines](https://tour.golang.org/concurrency/1) and [channels](https://tour.golang.org/concurrency/2). I encourage you to check them out if Go is something new for you.

So, now I have the access token. Now I need just to make the functionality for adding the songs 🙂

P.S. If you want to mess around with the Spotify API, remember first to [register](https://developer.spotify.com/dashboard/) your application to get the client id and secret.

<span>Photo by <a href="https://unsplash.com/@steve3p_0?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Steve Halama</a> on <a href="https://unsplash.com/s/photos/christmas-music?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>