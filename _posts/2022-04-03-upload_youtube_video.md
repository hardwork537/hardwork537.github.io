---
layout:     post
title:      通过api上传youtube视频
category:   golang
tags:       [youtube video]
description: 最近在做youtube视频的自动上传，从调研接口到demo完成也花了不少时间
---


## 文档

[Google Cloud Platform](https://console.cloud.google.com/apis/dashboard)

启用 api 和服务、申请凭据 等

[Youtube data api](https://developers.google.com/youtube/v3/docs/videos/list)

包括相关接口的调用方式及参数、可以实操运行

## 步骤

### 1. 申请 api 服务

首先你需要在自己创建的项目里面开启 google+ 和 youtube 两个 api

![google+ api](/images/youtube_video/google+_api.png)

![youtube data api](/images/youtube_video/youtube_api.png)

### 2. 创建 oauth 凭证

我们一共有两个身份凭据，一个是 API key，他的使用场景是：比如取得一些公开信息的时候，你是不需要用户授权访问的，访问 youtube 公开的 channel 就是这样

另外一个是 oauth2.0 认证，比如：视频上传视频、拉取个人视频等信息

![oauth2](/images/youtube_video/oauth2.png)

创建的时候应用类型可以选择 “web 应用”

![oauth2_create](/images/youtube_video/create_auth.png)

不过 web 类型需要提供一个重定向 URI，不过只需要可以调通返回 status：200 就可以

![callback_url](/images/youtube_video/callback_url.png)

### 3. 授权账号

在第一次操作时，需要通过 web 获取 token；然后把 token 保存到文件中，后面定期根据 refresh_token 来更新 token，后期基本就可以完全通过 API 操作了

第一次 web 操作时，需要对具体账号进行授权，才可以调用接口，所以要在 Oauth 同意屏幕中把账号填进去

![login_account](/images/youtube_video/login_account.png)

![oauth_agree](/images/youtube_video/oauth_agree.png)

### 4. 下载 json 文件

![down_json](/images/youtube_video/down_json.png)

## 代码

### upload_youtube_video

```
package main

import (
	"flag"
	"fmt"
	"log"
	"os"
	"strings"
	"time"

	"google.golang.org/api/youtube/v3"
)

var (
	filename    = flag.String("filename", "", "Name of video file to upload")
	title       = flag.String("title", "Test Title", "Video title")
	description = flag.String("description", "Test Description", "Video description")
	category    = flag.String("category", "22", "Video category")
	keywords    = flag.String("keywords", "", "Comma separated list of video keywords")
	privacy     = flag.String("privacy", "unlisted", "Video privacy status")
)

func main() {
	flag.Parse()

	if *filename == "" {
		log.Fatalf("You must provide a filename of a video file to upload")
	}

	client := getClient(youtube.YoutubeUploadScope)
	service, err := youtube.New(client)
	if err != nil {
		log.Fatalf("Error creating YouTube client: %v", err)
	}

	upload := &youtube.Video{
		Snippet: &youtube.VideoSnippet{
			Title:       *title,
			Description: *description,
			CategoryId:  *category,
		},
		Status: &youtube.VideoStatus{PrivacyStatus: *privacy},
	}

	// The API returns a 400 Bad Request response if tags is an empty string.
	if strings.Trim(*keywords, "") != "" {
		upload.Snippet.Tags = strings.Split(*keywords, ",")
	}
	startTime := time.Now()
	call := service.Videos.Insert([]string{"snippet", "status"}, upload)

	file, err := os.Open(*filename)
	defer file.Close()
	if err != nil {
		log.Fatalf("Error opening %v: %v", *filename, err)
	}

	response, err := call.Media(file).Do()
	duration := time.Since(startTime)
	fmt.Printf("duration:%+v\n", duration)
	//handleError(err, "")
	fmt.Printf("Upload successful! Video ID: %v\n", response.Id)
}
```

### oauth2

```
package main

import (
    "encoding/json"
    "fmt"
    "io/ioutil"
    "log"
    "net"
    "net/http"
    "net/url"
    "os"
    "os/exec"
    "os/user"
    "path/filepath"
    "runtime"

    "golang.org/x/net/context"
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

// This variable indicates whether the script should launch a web server to
// initiate the authorization flow or just display the URL in the terminal
// window. Note the following instructions based on this setting:
// * launchWebServer = true
//   1. Use OAuth2 credentials for a web application
//   2. Define authorized redirect URIs for the credential in the Google APIs
//      Console and set the RedirectURL property on the config object to one
//      of those redirect URIs. For example:
//      config.RedirectURL = "http://localhost:8090"
//   3. In the startWebServer function below, update the URL in this line
//      to match the redirect URI you selected:
//         listener, err := net.Listen("tcp", "localhost:8090")
//      The redirect URI identifies the URI to which the user is sent after
//      completing the authorization flow. The listener then captures the
//      authorization code in the URL and passes it back to this script.
// * launchWebServer = false
//   1. Use OAuth2 credentials for an installed application. (When choosing
//      the application type for the OAuth2 client ID, select "Other".)
//   2. Set the redirect URI to "urn:ietf:wg:oauth:2.0:oob", like this:
//      config.RedirectURL = "urn:ietf:wg:oauth:2.0:oob"
//   3. When running the script, complete the auth flow. Then copy the
//      authorization code from the browser and enter it on the command line.
const launchWebServer = false

const missingClientSecretsMessage = `
Please configure OAuth 2.0
To make this sample run, you need to populate the client_secrets.json file
found at:
%v
with information from the {{ Google Cloud Console }}
{{ https://cloud.google.com/console }}
For more information about the client_secrets.json file format, please visit:
https://developers.google.com/api-client-library/python/guide/aaa_client_secrets
`

// getClient uses a Context and Config to retrieve a Token
// then generate a Client. It returns the generated Client.
func getClient(scope string) *http.Client {
    ctx := context.Background()

    b, err := ioutil.ReadFile("client_secret.json")
    if err != nil {
        log.Fatalf("Unable to read client secret file: %v", err)
    }

    // If modifying the scope, delete your previously saved credentials
    // at ~/.credentials/youtube-go.json
    config, err := google.ConfigFromJSON(b, scope)
    if err != nil {
        log.Fatalf("Unable to parse client secret file to config: %v", err)
    }

    // Use a redirect URI like this for a web app. The redirect URI must be a
    // valid one for your OAuth2 credentials.
    //config.RedirectURL = "http://localhost:8090"
    // Use the following redirect URI if launchWebServer=false in oauth2.go
    //config.RedirectURL = "urn:ietf:wg:oauth:2.0:oob"

    cacheFile, err := tokenCacheFile()
    if err != nil {
        log.Fatalf("Unable to get path to cached credential file. %v", err)
    }
    tok, err := tokenFromFile(cacheFile)
    //err = fmt.Errorf("test")
    if err != nil {
        authURL := config.AuthCodeURL("state-token", oauth2.AccessTypeOffline)
        if launchWebServer {
            fmt.Println("Trying to get token from web")
            tok, err = getTokenFromWeb(config, authURL)
        } else {
            fmt.Println("Trying to get token from prompt")
            tok, err = getTokenFromPrompt(config, authURL)
        }
        if err == nil {
            saveToken(cacheFile, tok)
        }
    }
    return config.Client(ctx, tok)
}

// startWebServer starts a web server that listens on http://localhost:8080.
// The webserver waits for an oauth code in the three-legged auth flow.
func startWebServer() (codeCh chan string, err error) {
    listener, err := net.Listen("tcp", "localhost:8090")
    if err != nil {
        return nil, err
    }
    codeCh = make(chan string)

    go http.Serve(listener, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        code := r.FormValue("code")
        codeCh <- code // send code to OAuth flow
        listener.Close()
        w.Header().Set("Content-Type", "text/plain")
        fmt.Fprintf(w, "Received code: %v\r\nYou can now safely close this browser window.", code)
    }))

    return codeCh, nil
}

// openURL opens a browser window to the specified location.
// This code originally appeared at:
//   http://stackoverflow.com/questions/10377243/how-can-i-launch-a-process-that-is-not-a-file-in-go
func openURL(url string) error {
    var err error
    switch runtime.GOOS {
        case "linux":
        err = exec.Command("xdg-open", url).Start()
        case "windows":
        err = exec.Command("rundll32", "url.dll,FileProtocolHandler", "http://localhost:4001/").Start()
        case "darwin":
        err = exec.Command("open", url).Start()
        default:
        err = fmt.Errorf("Cannot open URL %s on this platform", url)
    }
    return err
}

// Exchange the authorization code for an access token
func exchangeToken(config *oauth2.Config, code string) (*oauth2.Token, error) {
    tok, err := config.Exchange(oauth2.NoContext, code)
    if err != nil {
        log.Fatalf("Unable to retrieve token %v", err)
    }
    return tok, nil
}

// getTokenFromPrompt uses Config to request a Token and prompts the user
// to enter the token on the command line. It returns the retrieved Token.
func getTokenFromPrompt(config *oauth2.Config, authURL string) (*oauth2.Token, error) {
    var code string
    fmt.Printf("Go to the following link in your browser. After completing "+
               "the authorization flow, enter the authorization code on the command "+
               "line: \n%v\n", authURL)

    if _, err := fmt.Scan(&code); err != nil {
        log.Fatalf("Unable to read authorization code %v", err)
    }
    fmt.Println(authURL)
    return exchangeToken(config, code)
}

// getTokenFromWeb uses Config to request a Token.
// It returns the retrieved Token.
func getTokenFromWeb(config *oauth2.Config, authURL string) (*oauth2.Token, error) {
    codeCh, err := startWebServer()
    if err != nil {
        fmt.Printf("Unable to start a web server.")
        return nil, err
    }

    err = openURL(authURL)
    if err != nil {
        log.Fatalf("Unable to open authorization URL in web server: %v", err)
    } else {
        fmt.Println("Your browser has been opened to an authorization URL.",
                    " This program will resume once authorization has been provided.\n")
        fmt.Println(authURL)
    }

    // Wait for the web server to get the code.
    code := <-codeCh
    return exchangeToken(config, code)
}

// tokenCacheFile generates credential file path/filename.
// It returns the generated credential path/filename.
func tokenCacheFile() (string, error) {
    usr, err := user.Current()
    if err != nil {
        return "", err
    }
    tokenCacheDir := filepath.Join(usr.HomeDir, ".credentials")
    os.MkdirAll(tokenCacheDir, 0700)
    return filepath.Join(tokenCacheDir,
                         url.QueryEscape("youtube-go.json")), err
}

// tokenFromFile retrieves a Token from a given file path.
// It returns the retrieved Token and any read error encountered.
func tokenFromFile(file string) (*oauth2.Token, error) {
    f, err := os.Open(file)
    if err != nil {
        return nil, err
    }
    t := &oauth2.Token{}
    err = json.NewDecoder(f).Decode(t)
    defer f.Close()
    return t, err
}

// saveToken uses a file path to create a file and store the
// token in it.
func saveToken(file string, token *oauth2.Token) {
    fmt.Println("trying to save token")
    fmt.Printf("Saving credential file to: %s\n", file)
    f, err := os.OpenFile(file, os.O_RDWR|os.O_CREATE|os.O_TRUNC, 0600)
    if err != nil {
        log.Fatalf("Unable to cache oauth token: %v", err)
    }
    defer f.Close()
    json.NewEncoder(f).Encode(token)
}
```

### refresh_token

```
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"golang.org/x/oauth2"
	"io/ioutil"
	"log"
)

func main() {
	err := refreshToken()
	fmt.Println("refreshToken result:", err)
}

func newConf() *oauth2.Config {
	b, err := ioutil.ReadFile("client_secret.json")
	if err != nil {
		log.Fatalf("Unable to read client secret file: %v", err)
	}
	type cred struct {
		ClientID     string   `json:"client_id"`
		ClientSecret string   `json:"client_secret"`
		RedirectURIs []string `json:"redirect_uris"`
		AuthURI      string   `json:"auth_uri"`
		TokenURI     string   `json:"token_uri"`
	}
	var j struct {
		Web       *cred `json:"web"`
		Installed *cred `json:"installed"`
	}
	if err = json.Unmarshal(b, &j); err != nil {
		log.Fatalf("json Unmarshal fail: %v", err)
	}

	return &oauth2.Config{
		ClientID:     j.Web.ClientID,
		ClientSecret: j.Web.ClientSecret,
		RedirectURL:  j.Web.RedirectURIs[0],
		Scopes:       []string{"scope1", "scope2"},
		Endpoint: oauth2.Endpoint{
			AuthURL:  j.Web.AuthURI,
			TokenURL: j.Web.TokenURI,
		},
	}
}

func refreshToken() error {
	cacheFile, err := tokenCacheFile()
	if err != nil {
		log.Fatalf("Unable to get path to cached credential file. %v", err)
	}
	tok, err := tokenFromFile(cacheFile)
	if err != nil {
		log.Fatalf("cacheFile not exit")
	}

	//ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	//	w.Header().Set("Content-Type", "application/json")
	//	w.Write([]byte(`{"access_token":"ACCESS_TOKEN",  "scope": "user", "token_type": "bearer", "refresh_token": "NEW_REFRESH_TOKEN"}`))
	//	return
	//}))
	//defer ts.Close()
	conf := newConf()
	tkr := conf.TokenSource(context.Background(), &oauth2.Token{RefreshToken: tok.RefreshToken})
	tk, err := tkr.Token()

	if err != nil {
		return nil
	}
	saveToken(cacheFile, tk)

	//if err != nil {
	//	t.Errorf("got err = %v; want none", err)
	//	return
	//}
	//if want := "NEW_REFRESH_TOKEN"; tk.RefreshToken != want {
	//	t.Errorf("RefreshToken = %q; want %q", tk.RefreshToken, want)
	//}
	return nil
}
```

