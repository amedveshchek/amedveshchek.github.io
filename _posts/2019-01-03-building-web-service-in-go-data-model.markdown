---
title: "Web-service in Go #1: Data Model"
layout: post
date: 2019-01-03 21:25
image: /assets/images/markdown.jpg
headerImage: false
tag:
- golang
- webservice
- postgres
category: blog
author: alexmedveshchek
description: "Web-service in Go #1: Data Model"
---

# Intro

Don't like to read long intros. The same to write. :expressionless: So let's move right to the point: we are about to go through the development process of a typical web-service from scratch without any web-frameworks, to understand details "under the hood". As an example we will develop a simple e-store. End-of-intro. :smirk:

[This is the complete code](https://gist.github.com/amedveshchek/aa17277078ce07fea3b71b672ae69217) for the article.

# Data model

Since we're going to create an e-commerce platform, we will expose a list of goods which we have to store somewhere. So we begin from creating a data structure `Product`:

{% highlight go %}
// models/product.go
package models

type Product struct {
    ID    int
    Name  string
    Price float64
}
{% endhighlight %}

Before writing code I prefer to think a bit about project's file structure. Ofcourse we can't to come up with complete structure, but we can try to follow best-practices, so for the moment the below structure seems to be appropriate:

```
main.go         # Driver code
models/         # Data models
    product.go      # Product-related things
    db.go           # Global DB driver
    dao.go          # Data Access Object
```

Okay. Now we're at the point when it's time to think about the datastorage. I prefer to use my own rule: in any incomprehensible situation use PosgreSQL. Ofcourse using `pq` driver for Go, which is quite good for any task when speaking about data access with Postgres. In go-code at first time we will use a `global-variable` approach to store a pointer to database driver. Such an approach we will improve in the next articles since it has several disadvantages, but for the moment it will be fine enough to see how it works and not to make code too complex:

{% highlight go %}
// models/db.go
package models

import (
    "database/sql"
    "log"

    _ "github.com/lib/pq"
)


var DB *sql.DB

func ConnectDB(connectionString string) {
    var err error
    DB, err = sql.Open("postgres", connectionString)
    if err != nil {
        log.Panic(err)
    }

    if err = DB.Ping(); err != nil {
        log.Panic(err)
    }
}
{% endhighlight %}

There are several important points that we have to mention:
- A `global-variable` approach used with variable `DB` is very convenient: 
  - `sql.DB` is concurrent-safe [by design](https://golang.org/pkg/database/sql/#DB);
  - typically we call `ConnectDB()` once somewhere in `main.go`, and then it's available for any other package.
- `sql.Open()` doesn't establish connection with database. [Consider](https://golang.org/src/database/sql/sql.go?s=20295:20352#L683) its code and you'll see that it just creates necessary data structures (connector etc). So, in order to establish actual connection we have to call `db.Ping()`.
- We imported `pq` as an anonymous package. Why? [Because](http://go-database-sql.org/importing.html) any Go-`sql`-driver registers itself using global method `database/sql/Register()` and its unique name. Any driver do it in its own `init()` function, and the same done in package `pq`. So we just have to include this package in order to run its `init()` function, but since we don't use anything from that package, include it anonymously to prevent go-compiler to bark at us.

OK now. In fact we're already have everything to access our data, it's simple. But let's create some data in our DB first:
- [install PostgreSQL](/simple-installation-of-postgresql) if you still haven't;
- add data:
{% highlight sql %}
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(6,2) NOT NULL
);

INSERT INTO products (name, price) VALUES
    ('Coffee', 15.67),
    ('Sugar', 2.99),
    ('Doughnuts', 4.33),
    ('Smartphone X', 149.99);
{% endhighlight %}

Quite good set of goods, huh? :smile: Now let's write a simple piece of code to access all this data:

{% highlight go %}
// main.go
package main

import (
    "fmt"
    "log"

    "estore/models"
)

func main() {
    models.ConnectDB("postgres://user:pass@localhost/estoredb")

    rows, err := models.DB.Query("SELECT * FROM products")
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()

    p := models.Product{}
    for rows.Next() {
        err := rows.Scan(&p.ID, &p.Name, &p.Price)
        if err != nil {
            log.Fatal(err)
        }
        fmt.Printf("%#v\n", p)
    }
    if err = rows.Err(); err != nil {
        log.Fatal(err)
    }
}
{% endhighlight %}

Let'r try it out:

```
estore$> go run main.go
&models.Product{ID:1, Name:"Coffee", Price:15.67}
&models.Product{ID:2, Name:"Sugar", Price:2.99}
&models.Product{ID:3, Name:"Doughnuts", Price:4.33}
&models.Product{ID:3, Name:"Smartphone X", Price:149.99}
```

It works! But to write sql-quries and all the `rows.Scan()`-stuff code each time turns to hell if we're going to develop the project in larger scale. Also code looks spacious and dirty: anyone who want's to understand what happens needs to read all the queries and code in details. Bad design.

So let's hide all the stuff under the hood of DAO (Data Access Object):

{% highlight go %}
// models/dao.go
package models

func GetAllProducts() ([]*Product, error) {
    rows, err := DB.Query("SELECT * FROM products")
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
    // rows.Next() doesn't return error - just `false`, so we have to check rows.Err()
    if err = rows.Err(); err != nil {
        return nil, err
    }

    return res, nil
}
{% endhighlight %}

Ofcourse our Data Access Object not looks as object at all for the moment. We'll fix in the next articles when improve our project. But take a look on the code of `main.go` - it became more neat obviously:

{% highlight go %}
// main.go
package main

import (
    "fmt"
    "log"

    "estore/models"
)

func main() {
    models.ConnectDB("postgres://user:pass@localhost/estoredb")

    products, err := models.GetAllProducts()
    if err != nil {
        log.Fatal(err)
    }

    for _, p := range products {
        fmt.Printf("%#v\n", p)
    }
}
{% endhighlight %}

Now we have all the template code of our data model -- we just need to push it forward to the level of ready e-store. Read on, the next article: [Web-service in Go #2: Handling HTTP](/building-web-service-in-go-handling-http).
