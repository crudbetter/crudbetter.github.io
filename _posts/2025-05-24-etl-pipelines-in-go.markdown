---
layout: post
title: "ETL pipelines in Go"
date: 2025-05-24 12:00:00 +0000
categories: go
---
I've been porting a service written for Node.js to Go. It's proving a useful exercise for exploring how to implement an ETL pipeline in Go, in particular how to leverage some of Go's concurrency features.

When complete, the service will extract data from a Postgres database, perform transformations on the data, write the output to an Excel spreadsheet and upload it to another service.

The domain is the logistics industry, so the data is about shipments. A trivial transformation could be to sum the total weight of the parcels in the shipment. The dataset could be very large (millions of shipments) so streaming is clearly important. As each shipment is read out of Postgres we want to transform, convert to Excel format and (multipart) upload as a pipeline. We don't want that amount of data in memory.

This can, of course, be achieved in Go without concurrency. So this post will start with a non-concurrent pipeline and build up to various versions of a concurrent pipeline. It won't delve into converting to Excel format or tackle multipart uploads. Instead, just focusing on extracting from Postgres, performing an arbitrary transformation and writing to a file on disk is enough to be instructive.

Let's get stuck in!

# Background

Before we look at any pipeline code, let's briefly set some background.

Postgres can be run in Docker with some shipments seeded using the `docker-entrypoint-initdb.d` volume as follows:

{% highlight plsql %}
CREATE TABLE Shipments(
  shipmentId bigint NOT NULL UNIQUE,
  consignmentNumber text NOT NULL,
  CONSTRAINT PK_Shipments PRIMARY KEY (shipmentId)
);

INSERT INTO Shipments(shipmentId, consignmentNumber)
SELECT n, 'consNum' || n FROM generate_series(1, 1000) n;
{% endhighlight %}

Each shipment retrieved from Postgres will be decoded to a simple struct:

{% highlight go %}
package model

type Shipment struct {
  ShipmentId        int
  ConsignmentNumber string
  TotalWeight       int
}
{% endhighlight %}

Note `totalWeight` isn't a column in the `Shipments` table. It will be a field derived during the pipeline transformation stage.

# Postgres access

There are two options for accessing Postgres in Go:

- the `database/sql` package in the standard library plus a driver
- a fully-fledged third-party library

`pgx` can be used for both options. Even though we won't be using Postgres specific features, the latter option is sensible for the sake of simplicity and because the service won't be targeting any other SQL databases.

I decided to write a package for all Postgres access, partly due to single responsibility concerns but primarily to keep non-pipeline code to a minimum in main.

There are two structs involved as follows:

{% highlight go %}
package pg

type Store struct {
  conn *pgx.Conn
}

type ShipmentsCursor struct {
  rows *pgx.Rows
  Err  error
}
{% endhighlight %}

Although `Store` and `ShipmentsCursor` are both exported, `conn` and `rows` can and should be kept internal to the `pg` package.

There is a function returning a new store. One of the benefits of using a package is that we can centralise any error wrapping.

{% highlight go %}
func NewStore(ctx context.Context) (_ Store, err error) {
  defer func() {
    if err != nil {
      err = fmt.Errorf("creating new postgres store: %w", err)
    }
  }()

  pgConn, err := pgx.Connect(ctx, os.Getenv("POSTGRES_URI"))
  if err != nil {
    return Store{}, err
  }

  err = pgConn.Ping(ctx)
  if err != nil {
    return Store{}, err
  }

  return Store{conn: pgConn}, nil
}
{% endhighlight %}

Next, we need a method on `Store` for getting a `ShipmentsCursor`. It looks too trivial here, but another benefit of using a package would become clearer as the complexity of the SQL query increases.

{% highlight go %}
func (store Store) GetShipmentsCursor(ctx context.Context) (ShipmentsCursor, error) {
  rows, err := store.conn.Query(ctx, "select shipmentId, consignmentNumber from Shipments")
  if err != nil {
    return ShipmentsCursor{}, fmt.Errorf("getting shipments cursor: %w", err)
  }

  return ShipmentsCursor{rows: &rows}, nil
}
{% endhighlight %}

To complete the first pass of implementing our `pg` package we need a method to get the next shipment from a cursor. This is very similar to the standard `database/sql` approach (just via `pgx`). There are other `pgx` specific approaches that could be taken.

{% highlight go %}
func (cursor *ShipmentsCursor) GetNextShipment(shipment *model.Shipment) bool {
  rows := *cursor.rows

  if !rows.Next() {
    cursor.Err = rows.Err()
    return false
  }

  if err := rows.Scan(&shipment.ShipmentId, &shipment.ConsignmentNumber); err != nil {
    cursor.Err = fmt.Errorf("getting next shipment: %w", err)
    return false
  }

  return true
}
{% endhighlight %}

And with that we can move on to pipeline code!

For the sake of brevity, all the following sections will omit the code for creating/getting and closing a `Store`, a `ShipmentsCursor` and an `os.File`. They will also omit error handling. Each section will simply discuss a function with the signature `func(pg.ShipmentsCursor, io.Writer)`.

# Non-concurrent pipeline

There's not a great deal to talk about here...

Memory usage is fine. As `pgCursor.GetNextShipment` takes a `*model.Shipment` the variable `nextShipment` is repeatably used and therefore takes up a single location in memory.

However, because we're processing a single shipment at a time, speed isn't going to be optimised. For some scenarios that may well be perfectably acceptable.

{% highlight go %}
func nonConcurrent(pgCursor pg.ShipmentsCursor, w io.Writer) {
  jsonEnc := json.NewEncoder(w)

  var nextShipment model.Shipment

  for pgCursor.GetNextShipment(&nextShipment) {
    nextShipment.TotalWeight = rand.Int()
    jsonEnc.Encode(nextShipment)
  }
}
{% endhighlight %}

# Concurrent pipeline #1 - non-waiting

The easiest way to get a bit of concurrency involved is to fire off some goroutines and let them at it.

We use a channel to pass shipments to them. Whichever goroutine finishes first will take the next shipment from the channel, therefore the ordering of shipments returned by Postgres is not preserved.

A bigger problem though, is that as soon as all shipments are read from Postgres the channel is closed and the pipeline completes. The effect is not all transformed shipments are guaranteed to be written to `w`. Indeed, during local testing, between 2 and 4 shipments are consistently lost.

{% highlight go %}
func concurrentNonWaiting(pgCursor pg.ShipmentsCursor, w io.Writer) {
  shipments := make(chan model.Shipment)

  jsonEnc := json.NewEncoder(w)

  for range 5 {
    go func() {
      for shipment := range shipments {
        shipment.TotalWeight = rand.Int()
        jsonEnc.Encode(shipment)
      }
    }()
  }

  var nextShipment model.Shipment

  for pgCursor.GetNextShipment(&nextShipment) {
    shipments <- nextShipment
  }

  close(shipments)
}
{% endhighlight %}

# Concurrent pipeline #2 - waiting

Of the two problems, the easiest to fix is the premature pipeline completion. We introduce a wait group and another goroutine to handle tidying up.

`wg.Wait` pauses the "main" goroutine until the wait counter is zero. The wait counter is decremented by `wg.Done` when each "pipeline" goroutine completes. Because we're using an unbuffered channel we know that the "postgres" goroutine can't complete until all the shipments it's written to the channel are read.

These features ensure an orderly goroutine clean up, and allow all transformed shipments to be written to `w`.

{% highlight go %}
func concurrentWaiting(pgCursor pg.ShipmentsCursor, w io.Writer) {
  var wg sync.WaitGroup
  wg.Add(5)

  shipments := make(chan model.Shipment)

  jsonEnc := json.NewEncoder(w)

  for range 5 {
    go func() {
      // "pipeline" goroutine 
      defer wg.Done()

      for shipment := range shipments {
        shipment.TotalWeight = rand.Int()
        jsonEnc.Encode(shipment)
      }
    }()
  }

  go func() {
    // "postgres" goroutine
    defer close(shipments)

    var nextShipment model.Shipment

    for pgCursor.GetNextShipment(&nextShipment) {
      shipments <- nextShipment
    }
  }()

  // "main" goroutine
  wg.Wait()
}
{% endhighlight %}

# Concurrent pipeline #3 - batches

As a reminder, the second problem to solve is maintaining the ordering of shipments returned by Postgres. As another reminder, we need to stream data, we can't hold all the transformed shipments in memory to then re-order them again.

The solution is to introduce batching to the pipeline. For a batch of shipments read from Postgres, they must be transformed and re-ordered before being written to `w`. That must all happen before the next batch is started.

There is a jump in complexity to achieve this. I found a [blog article][godev_pipelines] on go.dev seminal in understanding how to go about it. That article introduces two ideas:

1. Connecting each pipeline stage by channels (whereas so far we've only used one channel).
2. Having communication/notification channels.

Let's consider the three pipeline stages, which can be pulled out to their own functions and discussed seperately.

#### GetNextShipmentBatch

In the `pg` package we implement a new method to get the next shipment batch from a cursor. The postgres interaction is essentially the same as `GetNextShipment`, apart from we only retrieve `size` shipments. The concurrency features are familiar, making use of a `WaitGroup`, a goroutine for retrieving the shipments, and a goroutine for tidying up.

The main new thing to notice is the `done` channel. This is a notification channel. When the pipeline is processing the final batch and `Next` returns false, we can a send a notification. `struct{}{}` is essentially an "any", as the contents of the notification doesn't matter.

{% highlight go %}
func (cursor *ShipmentsCursor) GetNextShipmentBatch(size int, done chan<- any) <-chan model.Shipment {
  out := make(chan model.Shipment)

  var wg sync.WaitGroup
  wg.Add(1)

  go func() {
    defer wg.Done()

    for range size {
      hasNext := (*cursor.rows).Next()

      if !hasNext {
        cursor.Err = (*cursor.rows).Err()
        done <- struct{}{}
        return
      }

      var shipment model.Shipment

      if err := (*cursor.rows).Scan(&shipment.ShipmentId, &shipment.ConsignmentNumber); err != nil {
        cursor.Err = fmt.Errorf("getting next shipment batch: %w", err)
        done <- struct{}{}
        return
      }

      out <- shipment
    }
  }()

  go func() {
    wg.Wait()
    close(out)
  }()

  return out
}
{% endhighlight %}

#### transformBatch

`transformBatch` has very similar patterns to `GetNextShipmentBatch`. However, there is no notification channel passed, as `transformBatch` is a stage in the middle of the pipeline.

Although `shipment.TotalWeight = rand.Int()` is hardly a complicated transformation, having it pulled out to a pipeline stage illustrates how more involved transformations would be handled concurrently.

{% highlight go %}
func transformBatch(size int, in <-chan model.Shipment) <-chan model.Shipment {
  out := make(chan model.Shipment)

  var wg sync.WaitGroup
  wg.Add(size)

  for range size {
    go func() {
      defer wg.Done()

      for shipment := range in {
        shipment.TotalWeight = rand.Int()
        out <- shipment
      }
    }()
  }

  go func() {
    wg.Wait()
    close(out)
  }()

  return out
}
{% endhighlight %}

#### processBatch

Again, `processBatch` uses similar patterns to what we've already seen. There is another `done` notification channel, but this  time for notifying that the batch is fully processed.

{% highlight go %}
func processBatch(size int, jsonEnc *json.Encoder, in <-chan model.Shipment, done chan<- any) {
  var wg sync.WaitGroup
  wg.Add(1)

  go func() {
    defer wg.Done()

    shipments := make([]model.Shipment, 0, size)
    for shipment := range in {
      shipments = append(shipments, shipment)
    }
    sort.Slice(shipments, func(i, j int) bool {
      return shipments[j].ShipmentId > shipments[i].ShipmentId
    })
    for _, shipment := range shipments {
      jsonEnc.Encode(shipment)
    }
  }()

  go func() {
    wg.Wait()
    done <- struct{}{}
  }()
}
{% endhighlight %}

#### main

Finally, we get to the code running in the "main" goroutine! Again, we use a `WaitGroup` to make sure that the pipeline isn't exited prematurely. What is different is how we handle the goroutine exit conditions. We use a "for-select" to indefinitely loop and randomly execute one of the cases.

The `done` case/channel will only receive a notification when all shipments have been read from Postgres. We don't exit immediately at this point, as the last batch will still be executing. Instead we set a local flag, so that the next time the `batchDone` case/channel receives a notification we know we can safely exit.

{% highlight go %}
func concurrentBatches(pgCursor pg.ShipmentsCursor, w io.Writer) {
  batchSize := 5

  jsonEnc := json.NewEncoder(w)

  batchDone := make(chan any)
  done := make(chan any)

  var wg sync.WaitGroup
  wg.Add(1)

  go func() {
    defer func() {
      close(batchDone)
      close(done)
      wg.Done()
    }()

    isDone := false

    for {
      select {
      case <-batchDone:
        if isDone {
          return
        }

        shipmentBatch := pgCursor.GetNextShipmentBatch(batchSize, done)
        transformedBatch := transformBatch(batchSize, shipmentBatch)
        processBatch(batchSize, jsonEnc, transformedBatch, batchDone)
      case <-done:
        isDone = true
      }
    }
  }()

  batchDone <- struct{}{} // start

  wg.Wait()
}
{% endhighlight %}

# Conclusion

It's always easiest to learn something new when "real" requirements are involved. That's why porting a solved problem from another language is particularly instructive. You already understand the problem domain and get to focus on applying the idioms of another language to it. There's still plenty to solve with this problem domain in Go. I would like to add profiling in order to get a sense of the optimum batch size, i.e. memory consumption vs speed. There is also the handling of error scenarios to figure out, e.g. what happens if any of the pipeline stages error? How do you idiomatically handle errors from non-"main" goroutines? But, as far as this post goes, I hope you've found some of the above useful!

[godev_pipelines]: https://go.dev/blog/pipelines 