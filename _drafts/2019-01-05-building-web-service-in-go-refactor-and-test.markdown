---
title: "Web-service in Go #3: Refactor and Test!"
layout: post
date: 2019-01-05 15:33
image: /assets/images/markdown.jpg
headerImage: false
tag:
- golang
- webservice
- http
category: blog
author: alexmedveshchek
description: "Web-service in Go #3: Refactor and Test!"
---

Now we are going to make dramatical improvements of our service in order to convert it to more extensible and testable outlook. :neckbeard:

But at first we have to discuss several things.


## Dependency Injection (DI)

In fact there is a mess in meanings when using this term in development world (though didn't hear it in non-dev-world, huh :wink:):

1). The first meaning is injecting some dependency into the next "outside" layers in Uncle Bob's diagram, in order to make code more splitted and testable in the same time:
![CleanArchitecture](https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg)
In our case, we could not to use global variable `DB` and just pass a variable like `Env` to any method. That variable will hold any common thing like `Config` or `Logger` needed by numbers of packages:

```go
type Env struct {
      DB      *sql.DB
      Config  *Config
      Logger  *log.Logger
}
```

After that, typically we have two ways:

- either we implement all the logic in `Env`-methods, like this:

```go
// main.go
import "estore/env"

func main() {
    // ...
    e := env.NewEnv()
    http.HandleFunc(e.OnGetAllProducts)
    // ...
}

// env/env.go
func (e *Env) OnGetAllProducts(w http.ResponseWriter, r *http.Request) {
    ...
}
```

- or pass an instance of `Env` to every method in context-wise manner, which could be less readable, but allows to spread code among different packages:

```go
func main() {   
    // ...
    env := environment.NewEnv()

    http.HandleFunc(func(w http.ResponseWriter, r *http.Request) {
        handlers.OnGetAllProducts(env, w, r)
    })
    // ...
}
```

Both of this approaches allow to substitute `Env`-object with mock and to test it conveniently.


2). The second meaning is Dependecy Injection like in Java, meaning some machanism, like [Uber's dig](https://github.com/uber-go/dig), that does 2 things:
  - allows you to tell it about all objects you may need, typically describing constructors;
  - understands the parameters that your constructor needs and injects them to it.
    Like this:

```go
import (
    "estore/config"
    "estore/logger"
    "estore/models"
    "estore/handlers"

    "github.com/uber-go/dig"
)

func main() {
    // tell `dig` what kind of objects you have and how to construct them
    di := dig.New()

    di.Provide(config.NewConfig)
    di.Provide(logger.NewLogger)
    di.Provide(models.NewDAO)
    di.Provide(handlers.NewHandlers)

    // now just call any function with ready objects
    di.Invoke(func (dao models.DAO, hnd handler.Handlers) {
        // ... use any initialized object ...
    })
}
```

Typically such mechanism implements dependency graph, and decides which object should be constructed before which, using [topological sorting](https://en.wikipedia.org/wiki/Topological_sorting).
Also, in Go it uses reflection to determine types of constructors, their parameters, interfaces etc, that can work fast if initialize all the dependency graph on start, or slowly if it implements lazy approach.
And also, it resolves dependencies in runtime, so possible problems could be noticed only in runtime as well, especially with mentioned lazy-approach.

So here. Both approaches are related to each other, though, we won't speak about the latter type of Dependency Injection more in this series of articles. :grinning:


## Dependency Injection, again

In code below we implemented two main changes:
- dependency injection using environment structure, that injects to service and its related methods.
- interface-approach in order to test DAO.

Let's take a look at file structure:

```
main.go         # Driver code
env/
    env.go      # Environment
models/         # Data models
    product.go      # Product-related things
    db.go           # Global DB driver
    dao.go          # Data Access Object
service:        # HTTP-service code
    service.go      # service object
    home.go         # HTTP-handlers for home page
    products.go     # for products
```

Source code (also see [complete code](https://gist.github.com/amedveshchek/aaef8ce24f5fd40424e3918992adf93a) for the article):

```go
// main.go
package main

import (
    "log"

    "estore/env"
    "estore/models"
    "estore/service"
)

// todo: move it to config by yourself, aha? =)
const (
    httpPort     = 8080
    dbConnString = "postgres://user:pass@localhost/estoredb"
)

func main() {
    env := WireEnv(dbConnString)

    h := service.NewService(httpPort, env)
    err := h.Run()
    log.Fatal(err)
}

func WireEnv(dbConnString string) *env.Env {
    db, err := models.NewDB(dbConnString)
    if err != nil {
        log.Panic(err)
    }

    return &env.Env{
        DAO: models.NewDAO(db),
    }
}
```

```go
// env/env.go
package env

import "estore/models"

type Env struct {
    DAO models.DAO
}
```

```go
// service/service.go
package service

import (
    "fmt"
    "log"
    "net/http"

    "estore/env"
)

type service struct {
    port int
    mux  *http.ServeMux
}

func NewService(port int, e *env.Env) *service {
    mux := http.NewServeMux()

    // we use `inject()` wrapper to inject environment to each handler
    mux.HandleFunc("/", inject(onHome, e))
    mux.HandleFunc("/products/", inject(onGetAllProducts, e))

    return &service{
        port: port,
        mux:  mux,
    }
}

func inject(handler func(*env.Env, http.ResponseWriter, *http.Request), e *env.Env) func(w http.ResponseWriter, r *http.Request) {
    return func(w http.ResponseWriter, r *http.Request) {
        handler(e, w, r)
    }
}

func (s *service) Run() error {
    log.Printf("Listening on localhost:%d...", s.port)
    return http.ListenAndServe(fmt.Sprintf(":%d", s.port), s.mux)
}
```

Ofcourse, there is the only field `DAO` in `Env`. We didn't add any other things like `Config` and `Logger` for brevity, which is easy to add to `Env`, and then to get access to them from any place, wherever `Env` is used.

Also you may notice, we store `DAO` not by pointer. And that's why, DAO became interface now -- that will allow us to write mock:

```go
// models/dao.go
package models

type DAO interface {
    GetAllProducts() ([]*Product, error)
}

type dao struct {
    db *DB
}

// make sure `dao` implements interface, in compile time
var _ DAO = (*dao)(nil)

func NewDAO(db *DB) *dao {
    return &dao{db}
}

func (dao *dao) GetAllProducts() ([]*Product, error) {
    rows, err := dao.db.Query("SELECT * FROM products ORDER BY id")
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    res := []*Product{}
    for rows.Next() {
        p := &Product{}
        err := rows.Scan(&p.ID, &p.Name, &p.Price)
        if err != nil {
            return nil, err
        }
        res = append(res, p)
    }
    // rows.Next() doesn't return error - only `false`, so we have to check rows.Err()
    if err = rows.Err(); err != nil {
        return nil, err
    }

    return res, nil
}
```

`DAO` mock and service test:

```go
package service

import (
    "fmt"
    "io/ioutil"
    "net/http"
    "testing"

    "estore/env"
    "estore/models"
)

type daoMock struct{}

var _ models.DAO = (*daoMock)(nil)

func (dao *daoMock) GetAllProducts() ([]*models.Product, error) {
    res := []*models.Product{
        {ID: 3, Name: "Notebook Y", Price: 459.99},
        {ID: 5, Name: "Pencil", Price: 2.35},
    }

    return res, nil
}

func TestService(t *testing.T) {
    e := &env.Env{
        DAO: &daoMock{},
    }

    const testPort = 8088
    srv := NewService(testPort, e)
    go func() {
        assertNoError(srv.Run())
    }()

    resp, err := http.Get(fmt.Sprintf("http://localhost:%d/products/", testPort))
    assertNoError(err)
    body, err := ioutil.ReadAll(resp.Body)
    assertNoError(err)
    assertEqual("3 | Notebook Y | $459.99\n5 | Pencil | $2.35\n", string(body))
}

func assertNoError(err error) {
    if err != nil {
        panic(err)
    }
}

func assertEqual(expected, actual string) {
    if expected != actual {
        panic(fmt.Sprintf("'%s' != '%s'", expected, actual))
    }
}
```

[The complete code of the service](https://gist.github.com/amedveshchek/aaef8ce24f5fd40424e3918992adf93a).

All code snippets in this post are free to use under the [MIT Licence](https://opensource.org/licenses/MIT).
