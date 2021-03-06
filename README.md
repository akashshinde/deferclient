# Defer Panic client
[![GoDoc](https://godoc.org/github.com/deferpanic/deferclient?status.svg)](https://godoc.org/github.com/deferpanic/deferclient)

[![wercker status](https://app.wercker.com/status/b7a471949687969984843f7c5e5988a2/s "wercker status")](https://app.wercker.com/project/bykey/b7a471949687969984843f7c5e5988a2)

Defer Panic Client Lib.

 *  **Panic Handling** - Let deferclient catch and log any panic you get
    to your own dashboard.

 *  **Error Handling** - Handle your errrors - but log them.

 *  **HTTP latency** - DeferClient can log the latencies of all your hard
    hit http requests.

 *  **Micro Services Tracing** - DeferClient can log queries to other services so you know exactly what service is slowing down the initial request!

 *  **Database latency** - Get notified of slow database queries in your
    go app.

 *  **Metrics** - See:
    - go routines
    - memory
    - usage
    - gc
    - cgo

    and more automatically in your own dashboard.

 *  **Custom K/V** - Got something we don't support? You can log your own k/v metrics just as easily.


Translations:

* [简体中文](README_zh_cn.md)
* [Русский](README_ru_RU.md)

### Installation
``go get github.com/deferpanic/deferclient/deferstats``

**api key**

Get an API KEY via your shell or signup manually [here](https://deferpanic.com/signup):
```
 curl https://api.deferpanic.com/v1.9/users/create \
        -X POST \
        -d "email=test@test.com" \
        -d "password=password"
```

### HTTP Examples

Here we have 3 examples:
* log a fast request
* log a slow request
* log a panic

```go
package main

import (
	"fmt"
	"github.com/deferpanic/deferclient/deferstats"
	"net/http"
	"time"
)

func panicHandler(w http.ResponseWriter, r *http.Request) {
	panic("there is no need to panic")
}

func fastHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "this request is fast")
}

func slowHandler(w http.ResponseWriter, r *http.Request) {
	time.Sleep(3 * time.Second)
	fmt.Fprintf(w, "this request is slow")
}

func main() {
	dps := deferstats.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")

	go dps.CaptureStats()

	http.HandleFunc("/fast", dps.HTTPHandlerFunc(fastHandler))
	http.HandleFunc("/slow", dps.HTTPHandlerFunc(slowHandler))
	http.HandleFunc("/panic", dps.HTTPHandlerFunc(panicHandler))

	http.ListenAndServe(":3000", nil)
}
```

You can see that our slow request triggers our latency threshold and
winds up in our dashboard while the panic is also caught and winds up in
the dashboard.

The client works perfectly fine in non-HTTP applications:

### Log Errors
If you want to log your errors use this method. deferstats.Wrap
will log the bactrace and the error and ship it up to deferpanic
immediately.

```go
package main

import (
        "errors"
        "fmt"
        "github.com/deferpanic/deferclient/deferstats"
        "time"
)

var (
        dps *deferstats.Client
)

func errorTest() {
        err := errors.New("danger will robinson!")
        if err != nil {
                dps.Wrap(err)
                fmt.Println(err)
        }
}

func panicTest() {
        defer dps.Persist()
        panic("there is no need to panic")
}

func main() {
        dps = deferstats.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")

        errorTest()
        panicTest()

        time.Sleep(time.Second * 20)
}
➜ 
```

### Database Latency

```
package main

import (
  "database/sql"
  "github.com/deferpanic/deferclient/deferstats"
  _ "github.com/lib/pq"
  "log"
  "time"
)

func main() {
  dps := deferstats.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")

  _db, err := sql.Open("postgres", "dbname=dptest sslmode=disable")
  db := deferstats.NewDB(_db)

  go dps.CaptureStats()

  var id int
  var sleep string
  err = db.QueryRow("select 1 as num, pg_sleep(0.25)").Scan(&id, &sleep)
  if err != nil {
    log.Println("oh no!")
  }

  err = db.QueryRow("select 1 as num, pg_sleep(2)").Scan(&id, &sleep)
  if err != nil {
    log.Println("oh no!")
  }

  time.Sleep(120 * time.Second)
}
```

We have additional database ORM wrappers:

[gorp](https://github.com/deferpanic/dpgorp)
[sqlx](https://github.com/deferpanic/dpsqlx)

### Generic K/V
If you wish to log other k/v metrics this implements a very basic
counter over time.

```go
package main

import (
	"github.com/deferpanic/deferclient/deferkv"
	"time"
)

func main() {
	dkv := deferkv.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")

	dkv.Report("some_key", 10)

	dkv.Report("some_other_key", 30)

	time.Sleep(5 * time.Second)
}
```

### Micro-Services/SOA Tracing

Got a micro-services/SOA architecture? Now you can trace your queries
throughout and tie them back to the initial query that started it all!

Example

User Facing Service
```go
package main

import (
	"fmt"
	"github.com/deferpanic/deferclient/deferstats"
	"io/ioutil"
	"net/http"
	"net/url"
)

func handler(w http.ResponseWriter, r *http.Request) {

	// just pass your spanId w/each request
	resp, err := http.PostForm("http://127.0.0.1:7070/internal",
		url.Values{"defer_parent_span_id": {deferstats.GetSpanIdString(w)}})
	if err != nil {
		fmt.Println(err)
	}

	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)

	fmt.Fprintf(w, string(body))
}

func main() {
	dfs := deferstats.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")

	go dfs.CaptureStats()

	http.HandleFunc("/", dfs.HTTPHandlerFunc(handler))
	http.ListenAndServe(":9090", nil)
}
```

Slow Internal API
```go
package main

import (
	"encoding/json"
	"github.com/deferpanic/deferclient/deferstats"
	"net/http"
	"time"
)

type blah struct {
	Stuff string
}

func handler(w http.ResponseWriter, r *http.Request) {
	time.Sleep(250 * time.Millisecond)

	stuff := blah{
		Stuff: "some reply",
	}

	js, err := json.Marshal(stuff)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.Write(js)
}

func main() {
	dfs := deferstats.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")

	go dfs.CaptureStats()

	http.HandleFunc("/internal", dfs.HTTPHandlerFunc(handler))
	http.ListenAndServe(":7070", nil)
}
```

Notice that when you wrap your http handlers w/the deferstats
httphandler we instantly tie the front facing service to the slow
internal one.

### Set Environment
Want to monitor both staging and production? By default the environment
is set to 'production' but you can us a different environment just by
setting the environment variable to wahtever you wish and all data will
be tagged this way. Then you can change it from your settings in your
dashboard.

```go
package main

import (
	"github.com/deferpanic/deferclient/deferstats"
	"time"
)

func main() {
	dfs := deferstats.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")
	dfs.Setenvironment("some-other-environment")

	go dfs.CaptureStats()

	time.Sleep(120 * time.Second)
}
```

### Set AppGroup
Many deferPanic users are using micro-servіces/SOA and sometimes you
want to see activity from one application versus all of them.

You can set this via the AppGroup variable and then toggle it from your
settings in your dashboard.

```go
package main

import (
  "github.com/deferpanic/deferclient/deferstats"
  "time"
)

func main() {
  dfs := deferstats.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")
  dfs.SetappGroup("queueMaster3000")

  go dfs.CaptureStats()

  time.Sleep(120 * time.Second)
}
```

### Disable posting to deferpanic for test/dev environments
Pretty much everyone needs to disable the client in a dev/test mode
otherwise you are going to be overwhelmed with notifications.

You can disable it simply by using the setter function SetnoPost.

```go
package main

import (
  "github.com/deferpanic/deferclient/deferstats"
  "time"
)

func main() {
  dfs := deferstats.NewClient("v00L0K6CdKjE4QwX5DL1iiODxovAHUfo")
  dfs.SetnoPost(true)

  go dfs.CaptureStats()

  time.Sleep(120 * time.Second)
}
```

### Dependencies

There are currently no dependencies so this should work out of the box
for containers like docker && rocket.

### Documentation

See https://godoc.org/github.com/deferpanic/deferclient for documentation.

Defer Panic Client
