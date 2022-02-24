# A toy Spark framework built using parapet

This repo provides a guide on how to use a toy-spark framework built with parapet.

**Step 1**. Environment: Linux Ubuntu, openjdk-11, scala-2.13

**Step 2**. Download unzip [spark-worker](https://s3.amazonaws.com/parapet.io/spark-distribution/spark-worker-0.0.1-RC6.zip)

**Step 2.1**. (optional, required for __cluster__ mode) download [parapet-cluster](https://s3.amazonaws.com/parapet.io/cluster-distribution/parapet-cluster-0.0.1-RC6.zip)

**Step 3**. Create a new scala project with the following depenencies:

```
libraryDependencies += "io.parapet" %% "spark" % "0.0.1-RC6"
```

**Step 4**. Create main class `App.scala` in your package, for example: `io.parapet.spark.example` 


```scala
package io.parapet.spark.example

import cats.effect.IO
import io.parapet.core.Dsl.DslF
import io.parapet.core.Events
import io.parapet.net.Address
import io.parapet.spark.SparkType._
import io.parapet.spark.{Row, SchemaField, SparkContext, SparkSchema}
import io.parapet.{CatsApp, core}

object App extends CatsApp {

  import dsl._

  private val sparkSchema = SparkSchema(
    Seq(
      SchemaField("count", IntType),
      SchemaField("word", StringType)))


  def standalone: DslF[IO, SparkContext[IO]] = {
    SparkContext.builder[IO]
      .clusterMode(false)
      .workerServers(List(
        Address.tcp("localhost:5556"),
        Address.tcp("localhost:5557")))
      .build
  }

  class SimpleMap extends io.parapet.core.Process[IO] {
    override def handle: Receive = {
      case Events.Start =>
        for {
          sparkContext <- standalone
          inputDf <- sparkContext.createDataframe(Seq(
            Row.of(1, "THIS"),
            Row.of(2, "IS"),
            Row.of(3, "MAP"),
            Row.of(4, "DEMO"),
          ), sparkSchema)
          outDf <- inputDf.map(r => Row.of(r.getAs[Int](0) + 1, r.getAs[String](1).toLowerCase()))
          _ <- outDf.show
          sortedDf <- outDf.sortBy(_.getAs[Int](0))
          _ <- eval(println("Sorted df:"))
          _ <- sortedDf.show
        } yield ()
    }
  }

  override def processes(args: Array[String]): IO[Seq[core.Process[IO]]] = IO {
    Seq(new SimpleMap())
  }
}

```

Step build your project: `sbt package`

Copy the program jar from `{projectDir}/target/scala-2.13` to `./spark-worker-0.0.1-RC6/lib`


Step 6. create worker configs:

worker-1

```
id=worker-1
address=127.0.0.1:5556
```

worker-2

```
id=worker-2
address=127.0.0.1:5557
```

Step 7. Run workers

worker-1

```
./spark-worker-0.0.1-RC6/bin/spark-worker -c ./worker-1
00:40:08.862 [scala-execution-context-global-17] INFO worker-1-server - server[ref=worker-1-server, address=Address(tcp,*,5556)]: is listening...
```

worker-2

```
./spark-worker-0.0.1-RC6/bin/spark-worker -c ./worker-2
00:41:18.963 [scala-execution-context-global-15] INFO worker-2-server - server[ref=worker-2-server, address=Address(tcp,*,5557)]: is listening...
```

Step 8. Run the program either from IDE or from terminal (you would need to add jars from `./spark-worker-0.0.1-RC6/lib` to the program classpath).

Expected output:

```
00:50:18.540 [scala-execution-context-global-17] DEBUG worker-0 - client[id=worker-0] has been connected to Address(tcp,localhost,5556)
00:50:18.554 [scala-execution-context-global-17] DEBUG worker-1 - client[id=worker-1] has been connected to Address(tcp,localhost,5557)
00:50:20.669 [scala-execution-context-global-19] DEBUG io.parapet.spark.SparkContext - received mapResult[jobId=JobId(39befdc2-4f66-40e6-ab14-4503dfb394e3), taskId=TaskId(0c0a3928-d34d-4810-b117-d731fb237124)]
00:50:20.696 [scala-execution-context-global-19] DEBUG io.parapet.spark.SparkContext - received mapResult[jobId=JobId(39befdc2-4f66-40e6-ab14-4503dfb394e3), taskId=TaskId(4a15b71e-61fc-4f1a-a590-8e9e0f8e2954)]
+-------+------+
| count | word |
+-------+------+
|     4 |  map |
+-------+------+
|     5 | demo |
+-------+------+
|     2 | this |
+-------+------+
|     3 |   is |
+-------+------+
Sorted df:
+-------+------+
| count | word |
+-------+------+
|     2 | this |
+-------+------+
|     3 |   is |
+-------+------+
|     4 |  map |
+-------+------+
|     5 | demo |
+-------+------+

```


## Run in cluster mode

**Step-1**. Download parapet-cluster Step-2.1

**Step-2**. Create two cluster node config files

node-1.properties

```
node.id=node-1
node.address=127.0.0.1:4445
node.peers=node-2:127.0.0.1:4446
node.election-delay=10
node.heartbeat-delay=5
node.monitor-delay=10
node.peer-timeout=10
node.coordinator-threshold=0.8
```

node-2.properties

```
node.id=node-2
node.address=127.0.0.1:4446
node.peers=node-1:127.0.0.1:4445
node.election-delay=10
node.heartbeat-delay=5
node.monitor-delay=10
node.peer-timeout=10
node.coordinator-threshold=0.8
```

**Step 3**. Start cluster nodes

node-1: `./parapet-cluster-0.0.1-RC6/bin/cluster --config ./node-1.properties`
node-2: `./parapet-cluster-0.0.1-RC6/bin/cluster --config ./node-2.properties`

After awhile you should see the following lines in logs

```
2022-02-24 01:13:17 DEBUG LeaderElection:264 - current leader: localhost:4446
2022-02-24 01:13:20 DEBUG LeaderElection:264 - cluster state: ok
```

**Step 5**. Update worker configs, provide cluster servers. Add `worker.cluster-servers=127.0.0.1:4445,127.0.0.1:4446` to worker-1 and worker-2 configs

**Step 6**. Restart workers

You should see the following lines in worker nodes

```
01:21:32.084 [scala-execution-context-global-14] DEBUG io.parapet.cluster.node.NodeProcess - 127.0.0.1:4445 is leader
01:21:32.107 [scala-execution-context-global-14] DEBUG io.parapet.cluster.node.NodeProcess - joining group ''. attempts made: 0
01:21:32.412 [scala-execution-context-global-13] DEBUG io.parapet.cluster.node.NodeProcess - node has joined cluster group:
```


**Step 7**. Update `SparkContext` builder

```
  def cluster: DslF[IO, SparkContext[IO]] = {
    SparkContext.builder[IO]
      .address(Address.tcp("127.0.0.1:4444"))
      .workers(List("worker-1", "worker-2"))
      .clusterMode(true)
      .clusterServers(List(
        Address.tcp("127.0.0.1:4445"),
        Address.tcp("127.0.0.1:4446")))
      .build
  }
```

repace line `sparkContext <- standalone` with  `sparkContext <- cluster`

**Step 8**. Run program

Expected output

```
01:37:40.323 [scala-execution-context-global-12] DEBUG io.parapet.cluster.node.NodeProcess - get leader. attempts made: 0
01:37:42.266 [scala-execution-context-global-12] INFO driver-12124102926177-server - server[ref=driver-12124102926177-server, address=Address(tcp,*,4444)]: is listening...
01:37:42.276 [scala-execution-context-global-12] DEBUG tcp://127.0.0.1:4445 - client[id=driver-12124102926177] has been connected to Address(tcp,127.0.0.1,4445)
01:37:42.301 [scala-execution-context-global-12] DEBUG tcp://127.0.0.1:4446 - client[id=driver-12124102926177] has been connected to Address(tcp,127.0.0.1,4446)
01:37:42.755 [scala-execution-context-global-12] DEBUG io.parapet.cluster.node.NodeProcess - 127.0.0.1:4446 is leader
01:37:42.775 [scala-execution-context-global-12] DEBUG io.parapet.cluster.node.NodeProcess - joining group ''. attempts made: 0
01:37:43.119 [scala-execution-context-global-12] DEBUG io.parapet.cluster.node.NodeProcess - node has joined cluster group: 
01:37:43.256 [scala-execution-context-global-12] DEBUG io.parapet.cluster.node.NodeProcess - node id=worker-1, address=127.0.0.1:5556 has been created
01:37:43.484 [scala-execution-context-global-19] DEBUG io.parapet.cluster.node.NodeProcess - node id=worker-2, address=127.0.0.1:5557 has been created
01:37:43.919 [scala-execution-context-global-18] DEBUG io.parapet.spark.SparkContext - received mapResult[jobId=JobId(31b40d06-e6b9-4c54-b962-a7de94a2a687), taskId=TaskId(503f2ca7-6fcb-4e20-8699-f7fce9be8faa)]
01:37:44.304 [scala-execution-context-global-18] DEBUG io.parapet.spark.SparkContext - received mapResult[jobId=JobId(31b40d06-e6b9-4c54-b962-a7de94a2a687), taskId=TaskId(65990094-e073-411c-b054-6267ba570a71)]
+-------+------+
| count | word |
+-------+------+
|     2 | this |
+-------+------+
|     3 |   is |
+-------+------+
|     4 |  map |
+-------+------+
|     5 | demo |
+-------+------+
Sorted df:
+-------+------+
| count | word |
+-------+------+
|     2 | this |
+-------+------+
|     3 |   is |
+-------+------+
|     4 |  map |
+-------+------+
|     5 | demo |
+-------+------+
```






