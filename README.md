txtq - a small message queue for parallel processes
==============================================================

The `txtq` package helps parallel R processes send messages to each other. Let's say Process A and Process B are working on a parallel task together. First, both processes grab the queue.

``` r
path <- tempfile() # Define a path to your queue.
path # In real life, temp files go away when the session exits, so be careful.
#> [1] "/tmp/RtmpAZ5YvK/file2a6e46b040c6"
q <- txtq(path) # Create the queue.
```

The queue uses text files to keep track of your data.

``` r
list.files(q$path()) # The queue's underlying text files live in this folder.
#> [1] "db"    "head"  "lock"  "total"
q$list() # You have not pushed any messages yet.
#> [1] title   message
#> <0 rows> (or 0-length row.names)
```

Then, Process A sends instructions to Process B.

``` r
q$push(title = "Hello", message = "process B.")
q$push(
  title = c("Calculate", "Calculate"),
  message = c("sqrt(4)", "sqrt(16)")
)
q$push(title = "Send back", message = "the sum.")
```

You can inspect the contents of the queue from either process.

``` r
q$list()
#>       title    message
#> 1     Hello process B.
#> 2 Calculate    sqrt(4)
#> 3 Calculate   sqrt(16)
#> 4 Send back   the sum.
q$list(1) # You can specify the number of messages to list.
#>   title    message
#> 1 Hello process B.
q$count()
#> [1] 4
```

As Process A is pushing the messages, Process B can consume them.

``` r
q$pop(2) # If you pass 2, you are assuming the queue has >=2 messages.
#>       title    message
#> 1     Hello process B.
#> 2 Calculate    sqrt(4)
```

Those popped messages are not technically in the queue any longer.

``` r
q$list()
#>       title  message
#> 1 Calculate sqrt(16)
#> 2 Send back the sum.
q$count() # Number of messages technically in the queue.
#> [1] 2
```

But we still have a full log of all the messages that were ever sent.

``` r
q$log()
#>       title    message
#> 1     Hello process B.
#> 2 Calculate    sqrt(4)
#> 3 Calculate   sqrt(16)
#> 4 Send back   the sum.
q$total() # Number of messages that were ever queued.
#> [1] 4
```

Let's let Process B get the rest of the instructions.

``` r
q$pop() # q$pop() with no arguments just pops one message.
#>       title  message
#> 1 Calculate sqrt(16)
q$pop() # Call q$pop(-1) to pop all the messages at once.
#>       title  message
#> 1 Send back the sum.
```

Now let's say Process B follows the instructions in the messages. The last step is to send the results back to Process A.

``` r
q$push(title = "Results", message = as.character(sqrt(4) + sqrt(16)))
```

Process A can now see the results.

``` r
q$pop()
#>     title message
#> 1 Results       6
```

When you are done, you have the option to destroy the files in the queue.

``` r
q$destroy()
file.exists(q$path())
#> [1] FALSE
```

This entire time, the queue was locked when either process was trying to create, access, or modify it. That way, the results stay correct even when multiple processes try to read or change the data at the same time.

Similar work
============

liteq
-----

[Gábor Csárdi](https://github.com/gaborcsardi)'s [`liteq`](https://github.com/r-lib/liteq) package offers essentially the same functionality implemented with SQLite. It has a some additional features, such as the ability to detect crashed workers and re-queue failed messages, but it was in an early stage of development at the time `txtq` was released.

Other message queues
--------------------

There is a [plethora of message queues](http://queues.io/) beyond R, most notably [ZeroMQ](http://zeromq.org) and [RabbitMQ](https://www.rabbitmq.com/). In fact, [Jeroen Ooms](http://github.com/jeroen) and [Whit Armstrong](https://github.com/armstrtw) maintain [`rzmq`](https://github.com/ropensci/rzmq), a package to work with [ZeroMQ](http://zeromq.org) from R. Even in this landscape, `txtq` has advantages.

1.  Its user interface is incredibly friendly, and its internals are simple. No prior knowledge of sockets or message-passing is required.
2.  It is incredibly lightweight, R-focused, and easy to install. It only depends on R and a few packages on [CRAN](https://cran.r-project.org).
3.  Unlike socket-based technologies, `txtq` it is file-based.
    -   The queue persists even if your work crashes, so you can diagnose failures with `q$log()` and `q$list()`.
    -   Job monitoring is easy. Just open another R session and call `q$list()` while your work is running.
