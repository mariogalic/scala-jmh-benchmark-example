# How to benchmark Scala code?

This example project is my attempt at answering stackoverflow question [How to profile methods in Scala?][10]

----


The recommended approach to benchmarking Scala code is via [sbt-jmh][1] 

> "Trust no one, bench everything." - sbt plugin for JMH (Java
> Microbenchmark Harness)

This approach is taken by many of the major Scala projects, for example,

 - [Scala][2] programming language itself
 - [Dotty][3] (Scala 3)
 - [cats][4] library for functional programming
 - [Metals][5] language server for IDEs

Simple wrapper timer based on `System.nanoTime` is [not a reliable method][6] of benchmarking:

> `System.nanoTime` is as bad as `String.intern` now: you can use it,
> but use it wisely. The latency, granularity, and scalability effects
> introduced by timers may and will affect your measurements if done
> without proper rigor. This is one of the many reasons why
> `System.nanoTime` should be abstracted from the users by benchmarking
> frameworks

Furthermore, considerations such as [JIT warmup][7], garbage collection, system-wide events, etc. might [introduce unpredictability][8] into measurements:

> Tons of effects need to be mitigated, including warmup, dead code
> elimination, forking, etc. Luckily, JMH already takes care of many
> things, and has bindings for both Java and Scala.

Based on [Travis Brown's answer][9] here is an example of how to setup JMH benchmark for Scala

 1. Add jmh to `project/plugins.sbt`
    ```
    addSbtPlugin("pl.project13.scala" % "sbt-jmh" % "0.3.7")
    ```
 1. Enable jmh plugin in `build.sbt`
    ```
    enablePlugins(JmhPlugin)
    ```
 1. Add to `src/main/scala/bench/VectorAppendVsListPreppendAndReverse.scala`
    ```
    package bench

    import org.openjdk.jmh.annotations._

    @State(Scope.Benchmark)
    @BenchmarkMode(Array(Mode.AverageTime))
    class VectorAppendVsListPreppendAndReverse {
      val size = 1_000_000
      val input = 1 to size

      @Benchmark def vectorAppend: Vector[Int] = 
        input.foldLeft(Vector.empty[Int])({ case (acc, next) => acc.appended(next)})

      @Benchmark def listPrependAndReverse: List[Int] = 
        input.foldLeft(List.empty[Int])({ case (acc, next) => acc.prepended(next)}).reverse
    }
    ```
 1. Execute benchmark with 
    ```
    sbt "jmh:run -i 10 -wi 10 -f 2 -t 1 bench.VectorAppendVsListPreppendAndReverse"
    ```

The results are

```
Benchmark                                                   Mode  Cnt  Score   Error  Units
VectorAppendVsListPreppendAndReverse.listPrependAndReverse  avgt   20  0.024 ± 0.001   s/op
VectorAppendVsListPreppendAndReverse.vectorAppend           avgt   20  0.130 ± 0.003   s/op
```

which seems to indicate prepending to a `List` and then reversing it at the end is order of magnitude faster than keep appending to a `Vector`.


  [1]: https://github.com/ktoso/sbt-jmh
  [2]: https://github.com/scala/scala/blob/b6c6486f7105c76712bd68f5652af0cb0d4af908/project/plugins.sbt#L32
  [3]: https://github.com/lampepfl/dotty/blob/3d26c53f65d6967cc368f16a7899947d593cf08e/project/plugins.sbt#L13
  [4]: https://github.com/typelevel/cats/blob/424569f9c7dd7e99f6a6d1b72980db00b75627cb/project/plugins.sbt#L8
  [5]: https://github.com/scalameta/metals/blob/b025f2d4aa8b447366d6ee5e88817a45e6b9d03a/project/plugins.sbt#L2
  [6]: https://shipilev.net/blog/2014/nanotrusting-nanotime/
  [7]: https://stackoverflow.com/questions/36198278/why-does-the-jvm-require-warmup
  [8]: https://stackoverflow.com/a/22604185/5205022
  [9]: https://stackoverflow.com/a/59600699/5205022
  [10]: https://stackoverflow.com/questions/9160001/how-to-profile-methods-in-scala