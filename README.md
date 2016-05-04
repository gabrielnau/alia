# alia
[![Build Status](https://travis-ci.org/mpenet/alia.svg?branch=master)](https://travis-ci.org/mpenet/alia)

> Coan-Teen, the female death spirit who walks without feet.

Cassandra CQL3 client for Clojure wrapping [datastax/java-driver](https://github.com/datastax/java-driver).

## What's in the box?

* Built on an **extremely solid base**,
  [datastax/java-driver](https://github.com/datastax/java-driver),
  based on the new **CQL native protocol**
* **Simple API** with a minimal learning curve
* **Great performance**
* Provides an optional **versatile CQL3+ DSL**, [Hayt](#hayt-query-dsl)
* Support for **Raw queries**, **Prepared Statements** or **[Hayt](#hayt-query-dsl) queries**
* Can do both **Synchronous and Asynchronous** query execution
* Async interfaces using either **clojure/core.async** , simple
  *callbacks* or *manifold*
* Support for **all of
  [datastax/java-driver](https://github.com/datastax/java-driver)
  advanced options**: jmx, auth, SSL, compression, consistency, custom
  executors, custom routing and more
* Support and sugar for **query tracing**, **metrics**, **retry
  policies**, **load balancing policies**, **reconnection policies**
  and **UUIDs** generation
* Extensible **Clojure data types support** & **clojure.core/ex-data**
  integration
* **Lazy and potentialy chunked sequences over queries**
* Controled rows **streaming** via core.async channels
* First class support for cassandra **collections**, **User defined
  types**, that includes **nesting**.
* **Lazy row decoding by default**, but also **optional cheap user
  controled decoding via IReduce**

## Installation

### Alia

Please check the
[Changelog](https://github.com/mpenet/alia/blob/master/CHANGELOG.md) first
if you are upgrading. Alia runs on Clojure >= 1.7 (we're using IReduceInit internally)

Add the following to your dependencies:

```clojure
[cc.qbits/alia-all "3.1.5"]
```

This would include all the codec extensions and extra libraries.

You can also pick and choose what you need from alia's
[modules](https://github.com/mpenet/alia/tree/master/modules):

* [cc.qbits/alia](https://github.com/mpenet/alia/tree/master/modules/alia) This
  is the main module with all the basic alia functions (required).

* [cc.qbits/alia-async](https://github.com/mpenet/alia/tree/master/modules/alia-async):
  `clojure.core.async` interface to the core.async.

* [cc.qbits/alia-manifold](https://github.com/mpenet/alia/tree/master/modules/alia-manifold):
  `manifold` interface to manifold.

* [cc.qbits/alia-joda-time](https://github.com/mpenet/alia/tree/master/modules/alia-joda-time):
  Alia codec for joda-time types.

* [cc.qbits/alia-eaio-uuid](https://github.com/mpenet/alia/tree/master/modules/alia-eaio-uui):
  Alia codec for eaio-uuid library.

* [cc.qbits/alia-nippy](https://github.com/mpenet/alia/tree/master/modules/alia-nippy):
  Alia codec with nippy serialisation (for blobs)

### Hayt (the query DSL)

If you wish to use Hayt you need to add it to your dependencies

`[cc.qbits/hayt "3.0.1"]`

Then `require`/`use` `qbits.hayt` and you're good to go.

## Documentation

[codox generated documentation](http://mpenet.github.com/alia/#docs).

## Quickstart

Simple query execution using alia with hayt would look like this:

```clojure
(execute session (select :users
                         (where {:name :foo})
                         (columns :bar "baz")))
```


But first things first: here is an example of a complete session using
raw queries.


```clojure
(require '[qbits.alia :as alia])

(def cluster (alia/cluster {:contact-points ["localhost"]}))
```

Sessions are separate so that you can interact with multiple
keyspaces from the same cluster definition.

```clojure
(def session (alia/connect cluster))

(alia/execute session "CREATE KEYSPACE alia
                       WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};")
```



```clojure
   (alia/execute session "USE alia;")
   (alia/execute session "CREATE TABLE users (user_name varchar,
                                              first_name varchar,
                                              last_name varchar,
                                              auuid uuid,
                                              birth_year bigint,
                                              created timestamp,
                                              valid boolean,
                                              emails set<text>,
                                              tags list<bigint>,
                                              amap map<varchar, bigint>,
                                              PRIMARY KEY (user_name));")
  (alia/execute session "INSERT INTO users
                         (user_name, first_name, last_name, emails, birth_year, amap, tags, auuid,valid)
                         VALUES('frodo', 'Frodo', 'Baggins',
                         {'f@baggins.com', 'baggins@gmail.com'}, 1,
                         {'foo': 1, 'bar': 2}, [4, 5, 6],
                         1f84b56b-5481-4ee4-8236-8a3831ee5892, true);")

  ;; prepared statement with positional parameter(s)
  (def prepared-statement (alia/prepare session "select * from users where user_name=?;"))

  (alia/execute session prepared-statement {:values ["frodo"]})

  >> ({:created nil,
       :last_name "Baggins",
       :emails #{"baggins@gmail.com" "f@baggins.com"},
       :tags [4 5 6],
       :first_name "Frodo",
       :amap {"foo" 1, "bar" 2},
       :auuid #uuid "1f84b56b-5481-4ee4-8236-8a3831ee5892",
       :valid true,
       :birth_year 1,
       :user_name "frodo"})

  ;; prepared statement with named parameter(s)
  (def prepared-statement (alia/prepare session "select * from users where user_name= :name limit :lmt;"))

  (alia/execute session prepared-statement {:values {:name "frodo" :lmt 1}})


```

### Asynchronous interfaces:

There are currently 3 interfaces to use the asynchronous methods of
the underlying driver, the main one being **core.async**, another
using simple functions and an optional **manifold** interfaces is
also available.


### Async using function "callbacks"

```clojure
(execute-chan session
              "select * from users;"
              {:success (fn [rows] ...)
               :error (fn [e] ...)})
```

#### Async using clojure/core.async


`alia/execute-chan` has the same signature as the other execute
functions and as the name implies returns a clojure/core.async
promise-chan that will contain a list of rows at some point or an exception
instance.

Once you run it you have a couple of options to pull data from it.

+ using `clojure.core.async/take!` which takes the channel as first argument
and a callback as second:

```clojure
(take! (execute-chan session "select * from users;")
       (fn [rows-or-exception]
         (do-something rows)))
```

+ using `clojure.core.async/<!!` to block and pull the rows/exception
  from the channel.

```clojure
(def rows-or-exception (<!! (execute-chan session "select * from users;")))
```

+ using `clojure.core.async/go` block, and potentially using
  `clojure.core.async/alt!`.

```clojure
(go
 (loop [;;the list of queries remaining
        queries [(alia/execute-chan session (select :foo))
                 (alia/execute-chan session (select :bar))
                 (alia/execute-chan session (select :baz))]
        ;; where we'll store our results
        query-results '()]
   ;; If we are done with our queries return them, it's the exit clause
   (if (empty? queries)
     query-results
     ;; else wait for one query to complete (first completed first served)
     (let [[result channel] (alts! queries)]
       (println "Received result: "  result " from channel: " channel)
       (recur
        ;; we remove the channel that just completed from our
        ;; pending queries collection
        (remove #{channel} queries)

        ;; and finally we add the result of this query to our
        ;; bag of results
        (conj query-results result))))))
```

It also include `execute-chan-buffered`, which allows to run a single
query in a non-blocking way, returning a channel and streams the rows
in a controlled manner (respecting fetch-size and/or core async buffer
size) to it.

And it can do a lot more! Head to the
[codox generated documentation](http://mpenet.github.com/alia/#docs).

## Hayt (Query DSL)

There is a nicer way to write your queries using
[Hayt](https://github.com/mpenet/hayt), this should be familiar if you
know Korma or ClojureQL.
One of the major difference is that Hayt doesn't use macros and just
generates maps, so if you need to compose clauses or queries together
you can just use the clojure.core functions that work on maps.

Some examples:

```clojure

(use 'qbits.hayt)

(select :foo (where {:bar 2}))

;; this generates a map
>> {:select :foo :where {:bar 2}}

(update :foo
         (set-columns {:bar 1
                       :baz (inc-by 2)}
         (where [[= :foo :bar]
                 [> :moo 3]
                 [> :meh 4]
                 [:in :baz  [5 6 7]]]))


;; Composability using normal map manipulation functions

(def base (select :foo (where {:foo 1})))

(merge base
       (columns :bar :baz)
       (where {:bar 2})
       (order-by [:bar :asc])
       (using :ttl 10000))

;; To compile the queries just use ->raw

(->raw (select :foo))
> "SELECT * FROM foo;"

```

Alia supports hayt query direct execution, if you pass a non-compiled
query to `execute` or `execute-async`, it will be compiled and cached on a LU cache with a threshold of
100 (the cache function is user settable), so to be used carefully. The same is true with `prepare`.

Ex:
```clojure
(execute session (select :users (where {:name :foo})))
```

It covers everything that is possible with CQL3 (functions, handling
of collection types and their operations, ddl, prepared statements,
etc).
If you want to know more about it head to its [codox documentation](http://mpenet.github.com/hayt/codox/qbits.hayt.html) or
[Hayt's tests](https://github.com/mpenet/hayt/blob/master/test/qbits/hayt/core_test.clj).

## Mailing list

Alia has a
[mailing list](https://groups.google.com/forum/?fromgroups#!forum/alia-cassandra)
hosted on Google Groups.
Do not hesitate to ask your questions there.

## License

Copyright © 2013-2016 [Max Penet](https://twitter.com/mpenet)

Distributed under the Eclipse Public License, the same as Clojure.
