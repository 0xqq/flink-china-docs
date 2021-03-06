---
title:  批处理案例
nav-title: 批处理案例
nav-parent_id: examples
nav-pos: 20
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

下面的示例程序展示了Flink的不同应用程序，从简单的单词计数到图形算法。代码示例演示了[Flink的数据集API]({{ site.baseurl }}/dev/batch/index.html).
在Flink源代码仓库的{% gh_link flink-examples/flink-examples-batch "flink-examples-batch" %}模块中可以找到以下示例的完整源代码和更多示例。
* This will be replaced by the TOC
{:toc}


## 运行一个案例

为了运行Flink示例，我们假设您有一个正在运行的Flink实例可用。导航中的"Quickstart"和"Setup" 选项卡描述了启动Flink的各种方法。  
最简单的方法是运行`./bin/start-cluster.sh`集群。默认情况下，它使用一个JobManager和一个TaskManager启动本地集群。
Flink的每个二进制版本都包含一个`examples`目录，其中包含用于该页上每个示例的jar文件。  
要运行WordCount示例，发出以下命令:   
{% highlight bash %}
./bin/flink run ./examples/batch/WordCount.jar
{% endhighlight %}

其他示例也可以以类似的方式开始。


注意，通过使用内置数据，许多示例在运行时不传递任何参数。要使用真实数据运行WordCount，必须将路径传递给数据:  
{% highlight bash %}
./bin/flink run ./examples/batch/WordCount.jar --input /path/to/some/text/data --output /path/to/result
{% endhighlight %}

注意，非本地文件系统需要一个模式前缀，如`hdfs://`。
## Word Count 单词统计
WordCount是大数据处理系统的"Hello World"。它计算文本集合中单词的频率。该算法分为两个步骤:首先，将文本拆分为单个单词。其次，对单词进行分组和计数。  
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<String> text = env.readTextFile("/path/to/file");

DataSet<Tuple2<String, Integer>> counts =
        // split up the lines in pairs (2-tuples) containing: (word,1)
        text.flatMap(new Tokenizer())
        // group by the tuple field "0" and sum up tuple field "1"
        .groupBy(0)
        .sum(1);

counts.writeAsCsv(outputPath, "\n", " ");

// User-defined functions
public static class Tokenizer implements FlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
        // normalize and split the line
        String[] tokens = value.toLowerCase().split("\\W+");

        // emit the pairs
        for (String token : tokens) {
            if (token.length() > 0) {
                out.collect(new Tuple2<String, Integer>(token, 1));
            }   
        }
    }
}
{% endhighlight %}
{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/wordcount/WordCount.java  "WordCount example" %}使用输入参数实现上述算法:`--input <path> --output <path>`。作为测试数据，任何文本文件都可以。
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

// get input data
val text = env.readTextFile("/path/to/file")

val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
  .map { (_, 1) }
  .groupBy(0)
  .sum(1)

counts.writeAsCsv(outputPath, "\n", " ")
{% endhighlight %}

{% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/wordcount/WordCount.scala  "WordCount example" %}使用输入参数实现上述算法: `--input <path> --output <path>`。作为测试数据，任何文本文件都可以。

</div>
</div>

## Page Rank 网页排名  
PageRank算法计算链接定义的图中页面的"重要性"，链接从一个页面指向另一个页面。它是一种迭代图算法，即重复应用相同的计算。在每次迭代中，每个页面都将其当前的秩分布到所有相邻的页面上，并将其新秩计算为从相邻页面获得的秩的累加和。PageRank算法是由谷歌搜索引擎推广的，它利用网页的重要性对搜索查询结果进行排序。    
在这个简单的例子中，PageRank是通过[bulk iteration](iterations.html)和固定数量的迭代来实现的。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// read the pages and initial ranks by parsing a CSV file
DataSet<Tuple2<Long, Double>> pagesWithRanks = env.readCsvFile(pagesInputPath)
						   .types(Long.class, Double.class)

// the links are encoded as an adjacency list: (page-id, Array(neighbor-ids))
DataSet<Tuple2<Long, Long[]>> pageLinkLists = getLinksDataSet(env);

// set iterative data set
IterativeDataSet<Tuple2<Long, Double>> iteration = pagesWithRanks.iterate(maxIterations);

DataSet<Tuple2<Long, Double>> newRanks = iteration
        // join pages with outgoing edges and distribute rank
        .join(pageLinkLists).where(0).equalTo(0).flatMap(new JoinVertexWithEdgesMatch())
        // collect and sum ranks
        .groupBy(0).sum(1)
        // apply dampening factor
        .map(new Dampener(DAMPENING_FACTOR, numPages));

DataSet<Tuple2<Long, Double>> finalPageRanks = iteration.closeWith(
        newRanks,
        newRanks.join(iteration).where(0).equalTo(0)
        // termination condition
        .filter(new EpsilonFilter()));

finalPageRanks.writeAsCsv(outputPath, "\n", " ");

// User-defined functions

public static final class JoinVertexWithEdgesMatch
                    implements FlatJoinFunction<Tuple2<Long, Double>, Tuple2<Long, Long[]>,
                                            Tuple2<Long, Double>> {

    @Override
    public void join(<Tuple2<Long, Double> page, Tuple2<Long, Long[]> adj,
                        Collector<Tuple2<Long, Double>> out) {
        Long[] neighbors = adj.f1;
        double rank = page.f1;
        double rankToDistribute = rank / ((double) neigbors.length);

        for (int i = 0; i < neighbors.length; i++) {
            out.collect(new Tuple2<Long, Double>(neighbors[i], rankToDistribute));
        }
    }
}

public static final class Dampener implements MapFunction<Tuple2<Long,Double>, Tuple2<Long,Double>> {
    private final double dampening, randomJump;

    public Dampener(double dampening, double numVertices) {
        this.dampening = dampening;
        this.randomJump = (1 - dampening) / numVertices;
    }

    @Override
    public Tuple2<Long, Double> map(Tuple2<Long, Double> value) {
        value.f1 = (value.f1 * dampening) + randomJump;
        return value;
    }
}

public static final class EpsilonFilter
                implements FilterFunction<Tuple2<Tuple2<Long, Double>, Tuple2<Long, Double>>> {

    @Override
    public boolean filter(Tuple2<Tuple2<Long, Double>, Tuple2<Long, Double>> value) {
        return Math.abs(value.f0.f1 - value.f1.f1) > EPSILON;
    }
}
{% endhighlight %}

{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/graph/PageRank.java "PageRank program" %}实现了上面的示例。
它需要以下参数来运行:`--pages <path> --links <path> --output <path> --numPages <n> --iterations <n>`。
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
// User-defined types
case class Link(sourceId: Long, targetId: Long)
case class Page(pageId: Long, rank: Double)
case class AdjacencyList(sourceId: Long, targetIds: Array[Long])

// set up execution environment
val env = ExecutionEnvironment.getExecutionEnvironment

// read the pages and initial ranks by parsing a CSV file
val pages = env.readCsvFile[Page](pagesInputPath)

// the links are encoded as an adjacency list: (page-id, Array(neighbor-ids))
val links = env.readCsvFile[Link](linksInputPath)

// assign initial ranks to pages
val pagesWithRanks = pages.map(p => Page(p, 1.0 / numPages))

// build adjacency list from link input
val adjacencyLists = links
  // initialize lists
  .map(e => AdjacencyList(e.sourceId, Array(e.targetId)))
  // concatenate lists
  .groupBy("sourceId").reduce {
  (l1, l2) => AdjacencyList(l1.sourceId, l1.targetIds ++ l2.targetIds)
  }

// start iteration
val finalRanks = pagesWithRanks.iterateWithTermination(maxIterations) {
  currentRanks =>
    val newRanks = currentRanks
      // distribute ranks to target pages
      .join(adjacencyLists).where("pageId").equalTo("sourceId") {
        (page, adjacent, out: Collector[Page]) =>
        for (targetId <- adjacent.targetIds) {
          out.collect(Page(targetId, page.rank / adjacent.targetIds.length))
        }
      }
      // collect ranks and sum them up
      .groupBy("pageId").aggregate(SUM, "rank")
      // apply dampening factor
      .map { p =>
        Page(p.pageId, (p.rank * DAMPENING_FACTOR) + ((1 - DAMPENING_FACTOR) / numPages))
      }

    // terminate if no rank update was significant
    val termination = currentRanks.join(newRanks).where("pageId").equalTo("pageId") {
      (current, next, out: Collector[Int]) =>
        // check for significant update
        if (math.abs(current.rank - next.rank) > EPSILON) out.collect(1)
    }

    (newRanks, termination)
}

val result = finalRanks

// emit result
result.writeAsCsv(outputPath, "\n", " ")
{% endhighlight %}


{% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/graph/PageRankBasic.scala "PageRank program" %}实现上面的示例。
它需要以下参数来运行: `--pages <path> --links <path> --output <path> --numPages <n> --iterations <n>`.
</div>
</div>

输入文件是纯文本文件，必须格式化如下:  
- 页面表示为由新行字符分隔的(长)ID。
  * 例如`"1\n2\n12\n42\n63\n"`给出了5页的IDs 1、2、12、42和63。
- 链接表示为由空格字符分隔的页面id对。链接用换行符分隔:
    * 例如 `"1 2\n2 12\n1 12\n42 63\n"` 给四(直接)链接(1)->(2),(2)->(12),(1)->(12)和(42)->(63)。

对于这个简单的实现，要求每个页面至少有一个传入和一个传出链接(页面可以指向自己)。

## 连接组件

连通分量算法通过将同一连通部分中的所有顶点分配给相同的分量ID来识别一个较大图中的连通部分。与PageRank类似，连通分量是一种迭代算法。在每一步中，每个顶点将其当前的组件ID传播到它的所有邻居。如果一个顶点的组件ID小于它自己的组件ID，那么它接受来自邻居的组件ID。

此实现使用[delta iteration](iterations.html):没有更改其组件ID的顶点不参与下一步。这将产生更好的性能，因为后面的迭代通常只处理少数离群点。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
// read vertex and edge data
DataSet<Long> vertices = getVertexDataSet(env);
DataSet<Tuple2<Long, Long>> edges = getEdgeDataSet(env).flatMap(new UndirectEdge());

// assign the initial component IDs (equal to the vertex ID)
DataSet<Tuple2<Long, Long>> verticesWithInitialId = vertices.map(new DuplicateValue<Long>());

// open a delta iteration
DeltaIteration<Tuple2<Long, Long>, Tuple2<Long, Long>> iteration =
        verticesWithInitialId.iterateDelta(verticesWithInitialId, maxIterations, 0);

// apply the step logic:
DataSet<Tuple2<Long, Long>> changes = iteration.getWorkset()
        // join with the edges
        .join(edges).where(0).equalTo(0).with(new NeighborWithComponentIDJoin())
        // select the minimum neighbor component ID
        .groupBy(0).aggregate(Aggregations.MIN, 1)
        // update if the component ID of the candidate is smaller
        .join(iteration.getSolutionSet()).where(0).equalTo(0)
        .flatMap(new ComponentIdFilter());

// close the delta iteration (delta and new workset are identical)
DataSet<Tuple2<Long, Long>> result = iteration.closeWith(changes, changes);

// emit result
result.writeAsCsv(outputPath, "\n", " ");

// User-defined functions

public static final class DuplicateValue<T> implements MapFunction<T, Tuple2<T, T>> {

    @Override
    public Tuple2<T, T> map(T vertex) {
        return new Tuple2<T, T>(vertex, vertex);
    }
}

public static final class UndirectEdge
                    implements FlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {
    Tuple2<Long, Long> invertedEdge = new Tuple2<Long, Long>();

    @Override
    public void flatMap(Tuple2<Long, Long> edge, Collector<Tuple2<Long, Long>> out) {
        invertedEdge.f0 = edge.f1;
        invertedEdge.f1 = edge.f0;
        out.collect(edge);
        out.collect(invertedEdge);
    }
}

public static final class NeighborWithComponentIDJoin
                implements JoinFunction<Tuple2<Long, Long>, Tuple2<Long, Long>, Tuple2<Long, Long>> {

    @Override
    public Tuple2<Long, Long> join(Tuple2<Long, Long> vertexWithComponent, Tuple2<Long, Long> edge) {
        return new Tuple2<Long, Long>(edge.f1, vertexWithComponent.f1);
    }
}

public static final class ComponentIdFilter
                    implements FlatMapFunction<Tuple2<Tuple2<Long, Long>, Tuple2<Long, Long>>,
                                            Tuple2<Long, Long>> {

    @Override
    public void flatMap(Tuple2<Tuple2<Long, Long>, Tuple2<Long, Long>> value,
                        Collector<Tuple2<Long, Long>> out) {
        if (value.f0.f1 < value.f1.f1) {
            out.collect(value.f0);
        }
    }
}
{% endhighlight %}


{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/graph/ConnectedComponents.java "ConnectedComponents program" %}实现了上述示例。它需要以下参数来运行:`--vertices <path> --edges <path> --output <path> --iterations <n>`  

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
// set up execution environment
val env = ExecutionEnvironment.getExecutionEnvironment

// read vertex and edge data
// assign the initial components (equal to the vertex id)
val vertices = getVerticesDataSet(env).map { id => (id, id) }

// undirected edges by emitting for each input edge the input edges itself and an inverted
// version
val edges = getEdgesDataSet(env).flatMap { edge => Seq(edge, (edge._2, edge._1)) }

// open a delta iteration
val verticesWithComponents = vertices.iterateDelta(vertices, maxIterations, Array(0)) {
  (s, ws) =>

    // apply the step logic: join with the edges
    val allNeighbors = ws.join(edges).where(0).equalTo(0) { (vertex, edge) =>
      (edge._2, vertex._2)
    }

    // select the minimum neighbor
    val minNeighbors = allNeighbors.groupBy(0).min(1)

    // update if the component of the candidate is smaller
    val updatedComponents = minNeighbors.join(s).where(0).equalTo(0) {
      (newVertex, oldVertex, out: Collector[(Long, Long)]) =>
        if (newVertex._2 < oldVertex._2) out.collect(newVertex)
    }

    // delta and new workset are identical
    (updatedComponents, updatedComponents)
}

verticesWithComponents.writeAsCsv(outputPath, "\n", " ")

{% endhighlight %}


{% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/graph/ConnectedComponents.scala "ConnectedComponents program" %} 实现了上述示例。它需要以下参数来运行:`--vertices <path> --edges <path> --output <path> --iterations <n>`.
</div>
</div>

输入文件是纯文本文件，必须格式化如下:
- 顶点表示为id，用换行符分隔。
    * 例如`"1\n2\n12\n42\n63\n"`给了5个顶点(1)，(2)，(12)，(42)和(63)。
- 边缘表示为顶点id的对，顶点id由空间字符分隔。边缘用换行符分隔:
    * 例如`"1 2\n2 12\n1 12\n42 63\n"`给出4个(无向)链路(1)-(2)，(2)-(12)，(1)-(12)，和(42)-(63)。

{% top %}
