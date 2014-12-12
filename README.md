Scraping Reddit with Akka Streams 1.0
=====================================

> Reactive Streams is an initiative to provide a standard for asynchronous stream processing with non-blocking back pressure on the JVM.


Akka Streams provides a DSL for describing stream processing steps that are then materialized to create reactive streams implemented on top of Akka actors. In this post I describe the process of using akka-streams to fetch the most popular comments on reddit for some set of subreddits and persist word counts.

API Sketch:
-----------

```scala
type WordCount = Map[String, Int]
case class LinkListing(links: Seq[Link])
case class Link(id: String, subreddit: String)
case class CommentListing(subreddit: String, comments: Seq[Comment])
case class Comment(subreddit: String, body: String)

trait RedditAPI { // handles interaction with reddit's API,
  def popularLinks(subreddit: String)(implicit ec: ExecutionContext): Future[LinkListing]
  def popularComments(link: Link)(implicit ec: ExecutionContext): Future[CommentListing]
  def popularStrings(implicit ec: ExecutionContext): Future[Seq[String]]
}

trait KVStore { // in-memory key-value store
  def addWords(subreddit: String, words: WordCount): Future[Unit]
  def wordCounts: Future[Map[String, WordCount]]
}
```

Naive Solution:
--------------

Since our API functions yield futures, the simplest possible solution is to use flatMap and Future.sequence to chain together functions.

```scala
object SimpleExample {
  import RedditAPI._
  import ExecutionContext.Implicits.global

  def run =
    for {
      subreddits <- popularSubreddits
      linklistings <- Future.sequence(subreddits.map(popularLinks))
      links = linklistings.flatMap(_.links)
      commentListings <- Future.sequence(links.map(popularComments))
      comments = commentListings.flatMap(_.comments)
    } yield comments
}
```
This example fetches a list of subreddit names, issues simultaneous requests for the top links of each, then issues a request for comments. Try it for yourself: open up a console and type main.SimpleExample.run. You'll see a burst of requests which quickly start to fail with 503: service unavailable errors as Reddit's servers start to rate limit the connection.

Streams 101
-----------

[Scala DSL](http://doc.akka.io/api/akka-stream-and-http-experimental/1.0-M1/index.html#akka.stream.scaladsl.package): This domain specific language is used to create immutable stream descriptions that can be shared, composed, and materialized to create live streams composed of reactive stream primitives.
- Source: A Source is a set of stream processing steps that has one open output and an attached input. Can be used as a Publisher.
- Flow: A Flow is a set of stream processing steps that has one open input and one open output.
- Sink: A Sink is a set of stream processing steps that has one open input and an attached output. Can be used as a Subscriber.

[Reactive Stream Primitives](https://github.com/reactive-streams/reactive-streams): Streams built from these primitives are live streams, that can process elements and exert backpressure. They are what is created when Scala DSL entities are materialized.
- A Subscriber is a component that accepts a sequenced stream of elements provided by a Publisher.
- A Publisher is a provider of a potentially unbounded number of sequenced elements, publishing them according to the demand received from its Subscriber(s).


Streams Solution
-------------------

What we want is to issue requests at some regular interval. This will stop us from getting blocked for berserking their servers with thousands of requests in a short amount of time. First, we'll define a processing pipeline, then we'll construct a source of subreddits to connect it to.

First, we need a way to throttle a stream, such that it's limited to 1 message per time unit. We'll use the graph DSL to build a partial graph, a graph with a single undefined sink and source which can be used as a stream.

```scala
  def throttle[T](rate: FiniteDuration): Flow[T, T] = {
    val tickSource = TickSource(rate, rate, () => () )
    val zip = Zip[T, Unit] 
    val in = UndefinedSource[T]
    val out = UndefinedSink[T]
    PartialFlowGraph{ implicit builder =>
      import FlowGraphImplicits._
      in ~> zip.left  ~> Flow[(T,Unit)].map{ case (t, _) => t } ~> out
      tickSource ~> zip.right
    }.toFlow(in, out)
  }
```

This code constructs the following partial stream-processing graph

```
+------------+
| tickSource +-Unit-+
+------------+      +---> +-----+            +-----+      +-----+
                          | zip +-(T,Unit)-> | map +--T-> | out |
+----+              +---> +-----+            +-----+      +-----+
| in +----T---------+
+----+
````
Finally, the graph is converted to a flow from the vertex `in` to the vertex `out`.



Using throttle, we can now define a Flow[String, Comment] which handles all interactions with Reddit's API.

```scala
  val fetchComments: Flow[String, Comment] =
    // 0) Create a duct that applies no transformations.
    Flow[String]
        // 1) Throttle the rate at which the next step can receive subreddit names.
        .via(throttle)
        // 2) Fetch links. Subject to rate limiting.
        .mapAsyncUnordered( subreddit => RedditAPI.popularLinks(subreddit) )
        // 3) Flatten a stream of link listings into a stream of links.
        .mapConcat( listing => listing.links )
        // 4) Throttle the rate at which the next step can receive links.
        .via(throttle)
        // 5) Fetch links. Subject to rate limiting.
        .mapAsyncUnordered( link => RedditAPI.popularComments(link) )
        // 6) Flatten a stream of comment listings into a stream of comments.
        .mapConcat( listing => listing.comments )
```


We're also going to need to calculate word counts and write them to some store, ideally saving up comments to avoid an IO operation per comment. 

```scala
 val persistBatch: Flow[Comment, Int] =
    // 0) Create a duct that applies no transformations.
    Flow[Comment]
        // 1) Group comments, emitting a batch every 5000 elements
        //    or every 5 seconds, whichever comes first.
        .groupedWithin(5000, 5 second)
        // 2) Group comments by subreddit and write the wordcount
        //    for each group to the store. This step outputs
        //    the size of each batch after it is persisted for logging
        .mapAsyncUnordered{ batch =>
          val fs = batch
            .groupBy(_.subreddit)
            .mapValues(_.map(_.toWordCount).reduce(merge))
            .map{ case (subreddit, wordcount) =>
              store.addWords(subreddit, wordcount)
            }
          Future.sequence(fs).map{ _ => batch.size }
        }
```

So far, no processing has occurred. We've just described what we want to do. Now we create a starting flow of String names to which we append the Flow created in the previous steps, yielding a single `Source[Int]` the we can materialize and run.

```scala
def main(args: Array[String]): Unit = {
    // 0) Create a Source of String names, using either the provided
    //    argument vector or the result of the popularSubreddits API call.
    val subreddits: Source[String] =
      if (args.isEmpty)
        Source(RedditAPI.popularSubreddits).mapConcat(identity)
      else
        Source(args.toVector)

    // 1) Append flows to the initial source and materialize it via forEach.
    //    The resulting future succeeds if stream processing completes 
    //    or fails if an error occurs.
    val streamF: Future[Unit] =
      subreddits
        .via(fetchComments)
        .via(persistBatch)
        .foreach{ n => println(s"persisted $n comments")}

    // 2) When stream processing is finished, load the resulting
    //    word counts from the store, log some basic statistics,
    //    and write them to a .tsv files (code omited for brevity)
    ...
    }
```
