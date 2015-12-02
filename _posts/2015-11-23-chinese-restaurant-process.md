---
layout: post
title: "Optimized Chinese Restaurant Process"
subtitle: "or Clustering with the Dirichlet Process Mixture Model in Scala"
description: "Bayesian methods provide a theoretically well principled way to accomplish data science tasks, even basic tasks like clustering. Using a variety of performance optimizations we were able to sufficiently reduce the IO, memory and CPU (300,000&times;!) required to run large scale clustering based on the Chinese Restaurant Process (CRP). CRP is a non-parametric generative Bayesian model of a &quot;mixture&quot; that simultaneously learns the number of clusters, the model of each cluster, and entity assignments into clusters. We have open sourced this project in Scala for use on &quot;count&quot; data."
header-img: "img/mon-hands_grains.jpg"
authors:
    -
        name: "Ryan Richt"
        githubProfile: "ryan-richt"
        twitterHandle: "ryan_richt"
        avatarUrl: "https://avatars2.githubusercontent.com/u/541228?v=3"
tags: [scala, machine learning, bayesian, clustering]
---

## TL;DR

Bayesian methods provide a theoretically well principled way to accomplish data science tasks, even basic tasks like clustering. Using a variety of performance optimizations we were able to sufficiently reduce the IO, memory and CPU (300,000&times;!) required to run large scale clustering based on the Chinese Restaurant Process (CRP). CRP is a non-parametric generative Bayesian model of a "mixture" that simultaneously learns the number of clusters, the model of each cluster, and entity assignments into clusters. We have [open sourced](https://github.com/MonsantoCo/chinese-restaurant-process) this project in Scala for use on "count" data.

And you can run this sucker with:

```scala
import com.monsanto.stats.tables._
import com.monsanto.stats.tables.clustering._

val cannedAllTopicVectorResults: Vector[TopicVectorInput] = MnMGen.cannedData
val cannedCrp = new CRP(ModelParams(5, 2, 2), cannedAllTopicVectorResults)
val crpResult = cannedCrp.findClusters(200, RealRandomNumGen, cannedCrp.selectCluster)
```

>Iteration 1: cluster count was 365, reseat: 35, score: -29578.83920*  
>Iteration 2: cluster count was 118, reseat: 15, score: -29111.34349*  
>Iteration 3: cluster count was 61, reseat: 7, score: -28919.62995*  
>Iteration 4: cluster count was 40, reseat: 6, score: -28852.91482*  
>Iteration 5: cluster count was 29, reseat: 6, score: -28804.38123*  
>Iteration 6: cluster count was 24, reseat: 5, score: -28741.68993*  
>Iteration 7: cluster count was 16, reseat: 5, score: -28734.04974*  
>Iteration 8: cluster count was 14, reseat: 6, score: -28742.16624  
>Iteration 9: cluster count was 12, reseat: 5, score: -28739.19560  
>Iteration 10: cluster count was 10, reseat: 5, score: -28738.64498  
>...  
>Iteration 190: cluster count was 4, reseat: 10, score: -28724.77273  
>Iteration 191: cluster count was 3, reseat: 11, score: -28724.77273  
>Iteration 192: cluster count was 3, reseat: 10, score: -28724.77273  
>Iteration 193: cluster count was 3, reseat: 10, score: -28724.77273  
>Iteration 194: cluster count was 3, reseat: 10, score: -28724.77273  
>Iteration 195: cluster count was 3, reseat: 10, score: -28724.77273  
>Iteration 196: cluster count was 3, reseat: 10, score: -28724.77273  
>Iteration 197: cluster count was 3, reseat: 11, score: -28724.77273  
>Iteration 198: cluster count was 3, reseat: 10, score: -28724.77273  
>Iteration 199: cluster count was 3, reseat: 13, score: -28724.77273  
>Iteration 200: cluster count was 3, reseat: 12, score: -28724.77273


## Because who really knows what "K" should be anyway?

At Monsanto we have a variety of analytics and data science groups working on everything from sales transactions to aerial and satellite imaging to genome (DNA) sequencing. One of the oldest and most common data science problems is clustering: given a set of objects with possibly many properties, what is an appropriate partition of those entities into groups? Below we'll first describe the statistical method we used to perform clustering and then the software optimizations we implemented to make this scale. 

### Generative Bayesian Models

At Monsanto, we are Bayesians, and as the late [E.T. Jaynes](http://bayes.wustl.edu) espoused, we don't believe in "ad hockeries" like K-means (a numerical method) or ad hoc "machine learning" techniques such as random forests. Instead we have a better way: using only the laws of probability theory. Clustering is actually a difficult problem to cast in the Bayesian paradigm, but new theoretical results and the rise of computing power over the past few decades have made this problem tractable. 

Proper Bayesian models are "generative," meaning that they posit an underlying (or latent) "generating" process that creates the data we see. It is precisely writing a computer program to recreate the observed data, perhaps with some input variables missing that we want to recover. [Markov-chain Monte Carlo (MCMC)](https://en.wikipedia.org/wiki/Markov_chain_Monte_Carlo) then provides a universal mechanism to "invert" or solve for those input variables for such programs given some data. In the simplest case, say we observe the heights of a room of N people. The generating function could be, a normal distribution with some mean and variance that we draw N samples from. MCMC could then be run on those samples to try to recover the most likely parameters of that normal distribution.

We can also construct more complex generating functions (and still solve them with MCMC). Perhaps a better generating function would be to draw Male vs Female from a [binomial distribution](https://en.wikipedia.org/wiki/Binomial_distribution) (probability of being female in the population, like a weighted coin toss), and conditional on the result, draw a height from either the male-specific or female-specific height distribution.

### Bayesian Clustering

Mixture models are a Bayesian way of clustering: your generating function produces a mixed population of entities from an underlying discrete set of components. For instance, imagine I give you a stream of unlabeled bags of M&M's&#8482;<sup>[&dagger;](#mm)</sup> candies. All you get to observe is a few colored M&Ms™ of each bag. This is [multinomial](https://en.wikipedia.org/wiki/Multinomial_distribution) count data: we have a finite discrete "vocabulary" of colors and we will observe some number of counts of each color.

A multinomial distribution is just like a weighted (unfair) many-sided die, with one side for each outcome. For Christmas M&M's™ say we have a 3-sided die with faces indicating {Red, Green White} which lands on Red 40% of the time, Green 40% of the time and White 20% of the time. To generate a draw of size M (say, 35 candies) from this multinomial you just roll this die M times and count up a vector of each possible color.

![clustering colored candies](/img/MnMs_Clustering.png)

Imagine that the generating function is first buying a bag on a random day. Most of the year you can get bags with classic colors, for 2 weeks you can get Christmas colors, and for 3 days you can get 4th of July colors. So from this distribution of K types of bags say we draw N bags. Then we erase the packaging label of each, and I give you a handful of candies from each bag. Note that there is significant overlap in the colors from each kind of bag. From a small handful of say, Christmas M&M's™ where you didn't chance to draw and white colored candies, its hard to say if you they are Christmas or plain! From only these handfuls, your clustering job is to tell me:

1. How many kinds of bags there are
2. A model of the data produced from each kind of bag
3. Which handfuls came from which kinds of bags

So this is a clustering process. It is mixture of types of bags, and since we don't get the see the labels of the bags we have a mixture model. The unknown kinds of bags are the clusters, and the handfuls are our data. The multinomial counts of colors from the handfuls are our features extracted from the data, and we describe each type of bag by an explicit probability model which is a multinomial distribution.

Contrast this to say, K-means clustering:


| Attribute           | K-means                        | Bayesian Mixture Model         |
| ------------------- | ------------------------------ | ------------------------------ |
| Count of clusters   | Ad hoc - user specified "K"    | Probabilistic model            |
| Membership Measure  | Ad hoc - euclidean "distance"  | Probabilistic model            |
| Solution Method     | Ad hoc - EM*-style iterations  | Markov-chain Monte Carlo       |
| Confidence Measure  | None                           | Probabilities for all aspects  |


(*EM or [Expectation Maximization](https://en.wikipedia.org/wiki/Expectation–maximization_algorithm) is only guaranteed to converge to _local_ optima)

While K-means may work OK in some cases, it leaves much to be desired.

### The Chinese Restaurant Process

OK so we can describe a type of bag by a multinomial distribution. From several multinomials we can use [Bayes' Rule](https://en.wikipedia.org/wiki/Bayes%27_rule) to compute the relative probability that any given handful of candies belongs to each of the possible kinds of bags. But how do we posit kinds of bags in the first place? And how many might there be? The answer, and the probabilistic model for "choosing K" is the "Chinese Restaurant Process." The Chinese Restaurant Process is a generating function for a mixture model, and the story goes like this: 

![Chinese Restaurant Process](/img/Chinese_Restaurant_Setup.png)

There is a large family-style Chinese restaurant with a seemingly infinite number of infinitely large tables. A line of customers come in, and they join an existing table with probability proportional to how many others are already seated there (so popular tables get more popular), and with some probability they nucleate their own _new_ table. Every diner at the same table eats from the same dish, which is a common probability distribution. Their "bites" of the dish are our observed data points. So this is a generating function for a (clustering) mixture model, where we don't have to know K in advance and K can be unbounded!

The real beauty is that CRP properly probabilistically trades off between more "tighter" clusters and fewer more heterogenous clusters. Setting the "alpha" parameter determines the exchange rate of this trade-off, it doesn't specify K. You can think of the MCMC solver as running this generating function many times and looking for the highest probability assignments - where diners with similar "bites" are indeed assigned to the same table with the same dish, and we properly trade off the number of tables/dishes with how will the table-mates fit each dish. But instead we use a more efficient sort-of stochastic search that spends more time poking around high-probability regions but can still escape local maxima.

The precise low-down on the collapsed Gibbs sampler can best found [here](http://www.clsp.jhu.edu/~ajansen/miniws/Johnson11MLSS-talk-extras.pdf).

## The Optimization

We didn't set out to build our own implementation. Actually there is a great [series of DPMM/CRP/Clustering blog posts](http://blog.datumbox.com/the-dirichlet-process-the-chinese-restaurant-process-and-other-representations/) from the guys over at DatumBox, and that's where we stared. Open Source FTW!

Unfortunately we generated a large test data set with 100,000 "bags" each with Normal(400,100) "candies" sampled from 10,000 "colors" across 10 types of bags (clusters) with exponentially distributed membership, an Exponential(1/10) number of colors per type, and Exponential(1/100) weights of each color. Unfortunately, EXPLOSION!! And this explosion was reproducible on AWS on a behemoth memory optimized r3.8xlarge instance with a java heap size of 150GB!

Then we set out on what is a pretty archetypal optimization journey, but if you haven't done a lot of optimization, it may be of interest.

### Solve a Different Problem

It should also be noted that, we could have subsampled our data, implemented an approximation algorithm, or as we did do, solve another problem completely. MCMC is great for samples but according to [Daume 2007, _Fast search for Dirichlet process mixture models_](http://www.umiacs.umd.edu/~hal/docs/daume07astar-dp.pdf), it doesn't seem to be the most efficient search strategy if you only want the single most likely clustering. So (after we were unable to make the original Matlab work) we also reimplemented Daume 2007, which is a variant of A* search with some heuristics for this problem. Turns out even with substantial optimization, and a large beam (look back) size, we always got slower, worse clusterings than with our optimized Gibbs sampler. So it seemed the original problem was indeed the one worth solving.

### Memory and IO

The first thing we noticed was that the in-memory size of the data set was unnecessarily large. To keep Arrays of counts (aka, dense vectors) across 10,000 colors with Exponential(1/10) active colors meant that almost all of the data was zeros. While we love and make heavy use of [Breeze](https://github.com/scalanlp/breeze) we started out with the simplest thing that could possible work for a "sparse vector": a Map[Int, Int] from color index to count, filled in only for non-zero counts. This would require significant changes to the DatumBox code so we started over in Scala and implemented the collapsed Gibbs sampler for CRP with Dirichlet-Multinomial data in the [standard manner](http://www.clsp.jhu.edu/~ajansen/miniws/Johnson11MLSS-talk-extras.pdf).

This reduced our memory requirements from at least 150GB down to 2.2GB (68&times; RAM reduction), and improved startup time since we now needed only to parse in 6MB of data instead of 2000MB (~333&times; IO reduction).

![memory and IO reduction](/img/CRP_memory_IO_reduction.png)

### The CPU Saga

We were then very exciting to be able to run the Gibbs sampler in reasonable memory. Unfortunately we immediately hit the next problem: a single "reseating" of customers in the restaurant took 32.5 seconds. 32.5 seconds &times; 100,000 customers &times; 10,000 iterations = _1000 CPU years_. Ouch.

![CRP_cpu_time_reduction.png](/img/CRP_cpu_time_reduction.png)

Using a combination of the sampler and profiler in [VisualVM](https://visualvm.java.net/gettingstarted.html), manual timings, and micro-benchmarks we crafted a series of 7 versions that drove the reseating time down to 0.0001 seconds. Here are some of the highlights:

#### Initialization

If you read papers on CRP, you can see that there are numerous initialization strategies: 1 object per table, log N tables with random assignments, using _Daume 2007_ output as initialization, etc. While 1 object per table seems like the least likely to be biased based on random early decisions, it is also the slowest. We settled on random tables of 100 entities which gave us another 5&times; speed up or so, without detectable bias for our data.

#### Mutability

Of course we implemented the first version in the idiomatic, purely immutable Scala style, which involves a good deal of data structure copying. We first went whole hog changing everything to mutation and mutable data structures and indeed saw a ~10&times; speedup. Interestingly, it later turned out that we really only need to make a few data structures mutable (like [this one](https://github.com/MonsantoCo/chinese-restaurant-process/blob/4f0bcfaff0fdd78c8a17dd855fd89de4e34e690b/src/main/scala/com/monsanto/stats/tables/clustering/CRP.scala#L289)) and that they could be private to their respective methods or classes. As an example we used to have clusters/tables have a sequence of all of their members, and upon reseating a person we'd need to make a new cluster that was a copy of the old one with the new person. Instead it turned out to be much faster to both make that mutable and invert the order: members now have a mutable `Option[Cluster]` to which they currently belong and the cluster statistics are mutated upon add/remove. That's about a 5&times; speed up.

![add/remove occupant slow, refactor to mutable](/img/add_remove_occupant_slow.png) 

So the overall structure is something we call "immutable on the outside, mutable on the inside." Eventually we rewrote the codebase back to an immutable Scala style with only these few _private_ mutable arrays for performance. 

This is a great example of a classic lesson: there is usually only 1 hot path through the code. 90% of your code can remain pretty, idiomatic and immutable; only a small section needs to be uglified with optimization.

#### Caching

The collapsed Gibbs sampler for CRP has a central, tight, numerics heavy loop:

![slow estimate C function](/img/slow_estimate_c.png)


```scala
/*
 * C is just the result of this integral. C tells you the probability
 * that someone is going to sit somewhere and the probability of your
 * uncertainty about what the parameters of that table truly are. If you
 * toss 10 coins and get 6 heads 4 tails, you'd guess it is 60/40, but
 * you wouldn't be very certain. If you had 1000 samples you'd be more
 * certain, and likely be closer to 50/50. C it is accounting for that
 * uncertainty.
 */
def estimateCSmoothingFirst: Double = {

  // Compute partSumAi and partSumLogGammaAi by iterating through all
  // values in the WeightedVector's vecMap and computing the sum of the
  // values and their logGammas.
  // icky vars for performance in this critical path
  var partSumAi: Double = 0.0
  var partSumLogGammaAi: Double = 0.0
  var idx = 1
  val len = size * 2
  while (idx < len) {
    val v = pairs(idx)
    partSumAi += v + params.beta // add beta to this and the next value to smooth the curve
    val logGammaSmoothingFirst =
      if (v < allTopicVectorResults.length) cache(v)
      else logGamma(v + params.beta)
    partSumLogGammaAi += logGammaSmoothingFirst
    idx += 2
  }
```

And that logGamma special function, even given a numerical approximation expansion is pretty slow given all its instructions:

```scala
// Gamma is the continuous version of factorial, but off by 1, and its
// more accurate to compute it and its log in one step
def logGamma(x: Double): Double = {
  val tmp: Double = (x - 0.5) * Math.log(x + 4.5) - (x + 4.5)
  val ser: Double = 1.0 + 76.18009173 / (x + 0) - 86.50532033 / (x + 1) +
    24.01409822 / (x + 2) - 1.231739516 / (x + 3) +
    0.00120858003 / (x + 4) - 0.00000536382 / (x + 5)
  tmp + Math.log(ser * Math.sqrt(2 * Math.PI))
}
```
  
Turns out for our data set, this special function would literally be called _1 trillion times!_

In the naive implementation, we actually call logGamma on a `Double`. But that `Double` is really the value of the sum of some counts (an `Int`) and a prior probability term that certainly needs to be a `Double` because we often want values <1. So we pulled a couple of tricks:

* The value of that prior `Double` is constant for the whole run, it's not unknown, so what if we just add it _inside_ a caching function? Now it's a function of an `Int`.
* This is a function of positive `Int`'s over count data. So if we can assume some bounded size, there are a very small number of possible output values. Even if we allow the range 0-1,000,000 of input values, that's a tiny amount of memory and computes the function 1,000,000 times fewer! In fact we can just call the slow version if we're outside that range with a low-overhead if-check.
* We can even do better than using a `Map[Int, Double]`, since this is a function of Int (plus that double we add inside the function) we can just do direct lookups in an `Array` indexed by the argument.
* Turns out it's a lot of conditional logic and possible cache-blowing to check and fill in the map dynamically, we can just pre-fill the whole thing very fast on program startup. 

There's another 10&times;.

#### Boxing

The code has a number of `Map`'s and `Seq`'s of `Int`'s and `Double`'s which while normally innocuous once you get down to extreme optimization really start to add up with occasional un/boxing overhead. We fell in love with the open source library [Debox](https://github.com/non/debox) by Scala math genius Erik Asheim [@d6](https://twitter.com/d6) and recommend it highly. Subbing in these specialized data structures gave us another 5&times; speedup while keeping our code clean.

#### Micro-benchmarks: Fastest Map Combiner?

At some point the slowest part of the code was then computing the updated statistics for each table. These stats operate over the a (sparse) vector of counts summed across all diners sitting at this table. We have our now `Debox.Map` based sparse vectors, so what is the fastest way to sum a collection over them? Hint: its not "just the Monoid over addition!"

First we made a series of alternatives and timed this. Which do you think would be faster, or why are they not all the same?

```scala
val first = Range(1,30).map(i => (i -> 17)).toMap
val second = Range(15,44).map(i => (i -> 64.0)).toMap

def v1() = {
  // Add tv's vecMap to smoothedCounts vecMap. // MAKE THIS SIMPLER
  val temp: Vector[(Int, Double)] = first.toVector.map{
    case (i, j) => (i, j.toDouble)
  } ++ second.toVector
  val temp2: Map[Int, Vector[(Int, Double)]] = temp.groupBy(_._1)
  val temp3: Map[Int, Vector[Double]] = temp2.mapValues(xs => xs.map(_._2))
  val topicCountsSums: Map[Int, Double] = temp3.mapValues(_.sum)
  topicCountsSums.head
}

def v2() = {
  val allDenseKeys = first.keySet ++ second.keySet
  val diffs = allDenseKeys.map{ index =>
    index -> (first.getOrElse(index, 0) + second.getOrElse(index, 0.0))
  }.toMap
  diffs.head
}

def v3() = {

  val sums = mutable.Map.empty[Int, Double]
  first.keySet.foreach( k => sums(k) = sums.getOrElse(k, 0.0) + first(k) )
  second.keySet.foreach( k => sums(k) = sums.getOrElse(k, 0.0) + second(k) )
  sums.head
}

def v4() = {

  val sums = mutable.Map.empty[Int, Double]
  first.foreach{ kv => sums(kv._1) = sums.getOrElse(kv._1, 0.0) + kv._2 }
  second.foreach{ kv => sums(kv._1) = sums.getOrElse(kv._1, 0.0) + kv._2 }
  sums.head
}

def v5() = {

  val sums = mutable.Map.empty[Int, Double]
  first.foreach{ kv => val k = kv._1 ; sums(k) = sums.getOrElse(k, 0.0) + kv._2 }
  second.foreach{ kv => val k = kv._1 ; sums(k) = sums.getOrElse(k, 0.0) + kv._2 }
  sums.head
}
```

It turns out that after warmup, v1 is 2&times; as fast as v2, v3 is 2&times; as fast as v1, v4 is yet faster (but v5 is not). Who would have thought such big differences here!

Turns out, this is _still_ not the fastest way to do it. The slowest part comes in iterating over the lists twice because we need to compute the set of all the keys, or having to do the more expensive `getOrElse` calls. What if we could do everything in one pass? We settled on a class that implements are final version which makes use of the following facts:

* Our keys and values are both `Int`'s so we can keep them in one specialized `Array` as key1, value1, key2, value2, ... pairs to avoid lookups
* Even though its asymptotically more work, it's actually pretty low cost to keep the "maps" as _sorted_ lists of key value pairs (recall that say, hash tables have all O(1) operations and don't have to do any sorting)
* We can then _simultaneously_ iterate through both lists of key/value pairs and build up the summed sparse vector in one pass. If we are at the same key in both lists we can output their sum, if we are ahead on one side we know we can output the lower side to the output vector, and we can consume at different rates to ensure we're always in sync.

That looks about like this:

```scala
// Array(key0, value0, key1, value1, key2, value2, ...
// plus possibly some unused elements at the end)
final class VecMap private (private val pairs: Array[Int], val size: Int) {

  def +(that: VecMap): VecMap = {
    val thisLen = this.size * 2 // Length of used portion of this.pairs array
    val thatLen = that.size * 2 // Length of used portion of that.pairs array
    val newPairs: Array[Int] = new Array[Int](thisLen + thatLen)
    var thisIdx = 0
    var thatIdx = 0
    var newIdx = 0
    while (thisIdx < thisLen && thatIdx < thatLen) {
      val thisKey = this.pairs(thisIdx)
      val thatKey = that.pairs(thatIdx)
      if (thisKey == thatKey) {
        newPairs(newIdx) = thisKey
        newPairs(newIdx + 1) = this.pairs(thisIdx + 1) + that.pairs(thatIdx + 1)
        thisIdx += 2
        thatIdx += 2
      }
      else if (thisKey < thatKey) {
        newPairs(newIdx) = thisKey
        newPairs(newIdx + 1) = this.pairs(thisIdx + 1)
        thisIdx += 2
      }
      else {
        newPairs(newIdx) = thatKey
        newPairs(newIdx + 1) = that.pairs(thatIdx + 1)
        thatIdx += 2
      }
      newIdx += 2
    }
    if (thisIdx < thisLen) {
      // that.pairs is spent. Just finish off this
      while (thisIdx < thisLen) {
        newPairs(newIdx) = this.pairs(thisIdx)
        newPairs(newIdx + 1) = this.pairs(thisIdx + 1)
        thisIdx += 2
        newIdx += 2
      }
    }
    else if (thatIdx < thatLen) {
      // this.pairs is spent. Just finish off that
      while (thatIdx < thatLen) {
        newPairs(newIdx) = that.pairs(thatIdx)
        newPairs(newIdx + 1) = that.pairs(thatIdx + 1)
        thatIdx += 2
        newIdx += 2
      }
    }
    assert((newIdx & 1) == 0)
    new VecMap(newPairs, newIdx / 2)
  }
  ...
 }
```

#### Parallelization

Finally, of course this is Scala and we have very simple access to [parallel collections](http://docs.scala-lang.org/overviews/parallel-collections/overview.html). Interestingly here, Gibbs sampling is fundamentally sequential, but there are some opportunities for parallelism but several introductions of parallelism actually made CRP run slower! Always measure. But with the right use of parallel collections, such as computing the probabilities that a diner belongs to every possible existing table, we did get another 5&times; performance bump.

## Closing

We don't have the time to talk about every aspect of the couple weeks we spent squeezing a 300,000&times; speed improvement out of our naive CRP clustering implementation, but we hope some of the tools and strategies above might be useful in your work. At any rate, we hope you can make use of our JVM CRP implementation, which we believe to be the only JVM implementation available for large data sets, a foundational data science tool for clustering that we've now donated to the open source community.


<div style="font-size: 80%;">
<a name="mm"><sup>&dagger;</sup></a>M&M's&#8482; is a trademark of Mars, Inc used here for illustrative educational purposes.
</div>

<p></p>
