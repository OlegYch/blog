# Resolving OutOfMemory errors when running tests via sbt
Through the years i've seen many people complain about their tests suddenly crashing after a few runs. The usual reply is to use [forking](https://www.scala-sbt.org/1.x/docs/Forking.html). Since that kinda negates some of the benefits of sbt and is not a default behavior [https://github.com/sbt/sbt/issues/3983](https://github.com/sbt/sbt/issues/3983) i decided to dive in.

To be clear the issue is not sbt specific and affects anything that spawns multiple classloaders.
Java 8 and sbt 1.1.4 is assumed throughout the article.   

# Why
So why exactly you wouldn't want to just use forking by default? Here are a few reasons: 
- JVM startup

Not a big deal, probably 1 sec per test run, but adds up for each module.

- JIT

Again, probably not a big deal right now since newly loaded classes have to be jitted anyway, and sbt loads all of your classpath except java classes and scala-library [https://github.com/sbt/sbt/issues/4039](https://github.com/sbt/sbt/issues/4039) for each run. Most scala projects don't make heavy use of JVM classes and scala-library is usually just a tiny part of classpath.

Could probably have more effect if sbt is smarter with classloaders [https://github.com/sbt/sbt/issues/4111](https://github.com/sbt/sbt/issues/4111).
 
- Memory usage

By default JVM runs with practically unlimited heap (1/4 of your RAM), and if you have more than four modules running tests in parallel (or otherwise something else than the tests) that doesn't work very well. If enabling fork you need to always set `javaOptions` to something like `-Xmx200m`, though that still would waste memory. Reusing the jvm keeps memory usage limited and predictable.

Now, this all boils down to the same benefits and drawbacks that container servers like JBoss had, but in development environment the benefits far outweigh the drawbacks.

# How
Since Heap OOMEs are more or less well understood, and could be dealt with, i'm going to focus on Metaspace OOMEs. By default Metaspace is not limited at all which means you're going to eventually run out of system memory and depending on OS setup have your processes spill into pagefile (significantly slowing them down) or killed. If pagefile is not limited the same leaks will manifest itself as Heap OOMEs but probably much later. So for development you would want to modify `.sbtopts` file to something like this:
```
-J-XX:MinHeapFreeRatio=10
-J-XX:MaxHeapFreeRatio=20
-J-Xmx1g
-J-XX:MaxMetaspaceSize=600m
-J-XX:MetaspaceSize=200m
```
This limits heap and metaspace to a value that would allow both compilation and tests on a medium sized project. `MinHeapFreeRatio` and `MaxHeapFreeRatio` additionally allow jvm to give back Heap memory to OS more aggressively (eg after a full recompilation).

The reason Metaspace OOMEs are trickier to deal with is that garbage collection there happens only after references in Heap are collected, so if even a single reference is held by a class from a classloader all of the classes from that classloader are kept in memory. That means a reference to a single class can waste hundreds of megabytes of memory for a medium sized application.  

This [article](http://java.jiderhamn.se/2011/12/11/classloader-leaks-i-how-to-find-classloader-leaks-with-eclipse-memory-analyser-mat/) was very useful for detecting memory leaks leading to both Heap and Metaspace OOMEs. Here are a few reasons i found:
- threadpools

Lots of libs (eg `okhttp`, `web3j`) use a global threadpool which isn't ever going to be garbage collected (even if it's setup to use daemon threads), which means the class that started the threadpool won't be either. Your tests or apps will have to shutdown them manually:
```scala
    val executors = List(
      None -> classOf[org.web3j.utils.Async].getDeclaredField("executor"),
      None -> classOf[okhttp3.ConnectionPool].getDeclaredField("executor"),
    )
    executors.foreach {
      case (owner, f) =>
        f.setAccessible(true)
        f.get(owner.orNull).asInstanceOf[ExecutorService].shutdownNow()
    }
```
If you are passing `org.specs2.concurrent.ExecutionEnv` to your specs2 tests don't forget to shut it down after the test too, though not sure if it's still an issue in latest releases [https://github.com/etorreborre/specs2/issues/612](https://github.com/etorreborre/specs2/issues/612).

Scala global execution context seem to be okay to leave running since it's going to be reused.
- runaway threads

Some libs just spawn threads at will (again, `okhttp`) and leave them be, you can clean them up like this:
```scala
    Thread.getAllStackTraces.asScala.foreach {
      case (t, st) =>
        val classes = List("okio", "okhttp3", "Finalizer")
        if (classes.exists(c => st.exists(_.getClassName.contains(c)))) {
//          println(s"stopped ${t} at ${st.mkString("\n")}")
          t.stop()
        }
    }
```
- global references in jvm

If a library uses jvm classes and leaves a reference there, it won't be collected either, here is cleanup code for a few cases:
```scala
    java.sql.DriverManager.getDrivers.asScala.foreach(java.sql.DriverManager.deregisterDriver)
    val hooks = Class
      .forName("java.lang.ApplicationShutdownHooks")
      .getDeclaredField("hooks")
    hooks.setAccessible(true)
    hooks.get(null).asInstanceOf[java.util.IdentityHashMap[Thread, Thread]].clear()
``` 
- java.lang.ThreadLocal

This is a big PITA because the references are not going to be collected under normal circumstances and you can't clean it up (as far as i can tell):
```java
    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
``` 
You'll need to fix the libs that use it or revert to forking. Notable examples are `akka` and `play`. sbt itself is going to leak tests metaspace by default, but there is a workaround [https://github.com/sbt/sbt/issues/4112](https://github.com/sbt/sbt/issues/4112). 
- JVMTI

A curious case when attaching a profiler makes it hold on all classloaders (at least with JVisualVM). Make sure you have no profiler attached when debugging Metaspace leaks.

There are way more [ways](http://java.jiderhamn.se/2012/02/26/classloader-leaks-v-common-mistakes-and-known-offenders/) to trip up Metaspace GC, but thankfully they are much less relevant in scala world than they are in java.


Finally, since Metaspace GC depends on Heap GC, executing a bunch of runs immediately one after another like `;test;test;test;test;test;test` may blow up Metaspace even if there are technically no leaks, because Heap is large enough to not run a GC cycle.

# Addendum
There is a [library](https://github.com/mjiderhamn/classloader-leak-prevention/tree/master/classloader-leak-prevention/classloader-leak-prevention-core) which automates much of cleanup code. 
In the future i'll try to hook it up to sbt, meanwhile here is some example source code:

`.sbtopts`
```
-J-XX:MinHeapFreeRatio=10
-J-XX:MaxHeapFreeRatio=20
-J-Xmx1g
-J-XX:MaxMetaspaceSize=600m
-J-XX:MetaspaceSize=200m
```
`build.sbt`
```scala
lazy val root = project.aggregates(api, `api-it`)  
lazy val api = project
  .settings(
    Test / parallelExecution := true,
    Test / fork := false,
    Test / testOptions += Tests.Cleanup(loader => loader.loadClass("BaseTest").getMethod("cleanup").invoke(null)),
    )
  .disablePlugins(plugins.JUnitXmlReportPlugin)
//extract tests which depend on akka to a separate project  
lazy val `api-it` = project
  .settings(
    Test / parallelExecution := true,
    Test / fork := true,
    Test / javaOptions += "-Xmx200m",
  )
  .dependsOn(api % "test->test")
  .disablePlugins(plugins.JUnitXmlReportPlugin)
```
`api/src/test/scala/BaseTest.scala`
```scala
import java.util
import java.util.concurrent.ExecutorService

import org.specs2.concurrent.ExecutionEnv
import org.specs2.mutable.Specification
import org.specs2.specification.AfterAll

object BaseTest {
  //cleanup references to tests classloader to allow it be gced when running without fork
  def cleanup(): Unit = {
    import scala.collection.JavaConverters._
    java.sql.DriverManager.getDrivers.asScala.foreach(java.sql.DriverManager.deregisterDriver)
    val hooks = Class
      .forName("java.lang.ApplicationShutdownHooks")
      .getDeclaredField("hooks")
    hooks.setAccessible(true)
    hooks.get(null).asInstanceOf[util.IdentityHashMap[Thread, Thread]].clear()
    val executors = List(
      None -> classOf[org.web3j.utils.Async].getDeclaredField("executor"),
      None -> classOf[okhttp3.ConnectionPool].getDeclaredField("executor"),
//      Some(ExecutionContext.Implicits.global) -> ExecutionContext.Implicits.global.getClass.getDeclaredField("executor"),
    )
    executors.foreach {
      case (owner, f) =>
        f.setAccessible(true)
        f.get(owner.orNull).asInstanceOf[ExecutorService].shutdownNow()
    }
    Thread.getAllStackTraces.asScala.foreach {
      case (t, st) =>
        val classes = List("okio", "okhttp3", "Finalizer")
        if (classes.exists(c => st.exists(_.getClassName.contains(c)))) {
//          println(s"stopped ${t} at ${st.mkString("\n")}")
          t.stop()
        }
    }
  }
}

abstract class BaseTest extends Specification with AfterAll {
  protected implicit def ee: ExecutionEnv

  def afterAll(): Unit = {
    ee.shutdown()
  }
}
```
