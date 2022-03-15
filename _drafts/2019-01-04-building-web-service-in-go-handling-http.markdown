---
title: "Web-service in Go #2: Handling HTTP"
layout: post
date: 2019-01-04 23:12
image: /assets/images/markdown.jpg
headerImage: false
tag:
- golang
- webservice
- http
category: blog
author: alexmedveshchek
description: "Web-service in Go #2: Handling HTTP"
---

In [previous article](/building-web-service-in-go-data-model), we developed a complete code to access our data, which is also convenient for further refinements.

This is the next development stage to make the code be able to handle HTTP-requests:

```go
// main.go
package main

import (
    "fmt"
    "log"
    "net/http"

    "estore/models"
)

const port = "8080"

func main() {
    models.ConnectDB("postgres://user:pass@localhost/estoredb")

    http.HandleFunc("/products/", onGetAllProducts)

    log.Printf("Listening on localhost:%s...", port)
    http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
}

func onGetAllProducts(w http.ResponseWriter, r *http.Request) {
    if r.Method != "GET" {
        http.Error(w, http.StatusText(http.StatusMethodNotAllowed), http.StatusMethodNotAllowed)
        return
    }

    products, err := models.GetAllProducts()
    if err != nil {
        log.Printf("Error: %v", err)
        http.Error(w, http.StatusText(http.StatusInternalServerError), http.StatusInternalServerError)
        return
    }

    for _, p := range products {
        _, err := fmt.Fprintf(w, "%d | %s | $%.2f\n", p.ID, p.Name, p.Price)
        if err != nil {
            log.Printf("Error: %v", err)
            http.Error(w, http.StatusText(http.StatusInternalServerError), http.StatusInternalServerError)
            return
        }
    }
}
```

There are several important things that worse to mention:
- `http.HandleFunc()` is the method of default [ServeMux](https://golang.org/pkg/net/http/#ServeMux) which is defined globally in `http`-package. In fact it's **not** recommended to use default ServeMux: since [`DefaultServeMux`](https://godoc.org/net/http#DefaultServeMux) is global variable, any other package can add there anything it wants. In best case you'll get conflicting handlers, in worse case, package could be compromised and publish its own endpoint for further attacks on your service. So we're going to fix it below.
- Go's ServeMux doesn't support REST-routing. There is no templates/regexps allowed in routing paths like `/products/:id` to extract `id` from request.
- Take a look at `/products/` path -- it's with trailing slash. For the http-router it really has big meaning:
  - if path ends with slash `/`, then it will catch all the requests to any non-handling paths starting with `/products/...`.
  - the same story with the root `/` handler -- it will "catch" **all** the non-handling requests. For example, if user went to non-existent path `/brushes`, then it will be redirected to the root `/`. So it's advisible to check it in root handler, like this:
  ```go
  func onRoot(w http.ResponseWriter, r *http.Request) {
      if r.URL.Path != "/" {
          http.Error(w, "not found", 404)
          return
      }
      ...
  }
  ```

Now, in order to better organize code, let's put all handlers into a separate package, so that our file structure will look like this:

```
main.go         # Driver code
models/         # Data models
    product.go      # Product-related things
    db.go           # Global DB driver
    dao.go          # Data Access Object
service:
    products.go # HTTP handlers
```

Now, after all corrections and fixes the code of `main.go` will become:
```go
package main

import (
    "fmt"
    "log"
    "net/http"

    "estore/service"
    "estore/models"
)

const port = "8080"

func main() {
    models.ConnectDB("postgres://user:pass@localhost/estoredb")

    // don't use global ServeMux
    mux := http.NewServeMux()
    mux.HandleFunc("/products", service.OnGetAllProducts)

    log.Printf("Listening on localhost:%s...", port)
    if err := http.ListenAndServe(fmt.Sprintf(":%s", port), mux); err != nil {
        panic(err)
    }
}

```

Let's try it out:

```bash
estore$> go run main.go
Listening on localhost:8080...
```

Go to browser:

![try it out in browser](/assets/images/building-web-service-in-go-handling-http-browser.png)

In the [next article](/building-web-service-in-go-refactor-and-test) we will improve the code of our service in order it will become more extensible and testable.
