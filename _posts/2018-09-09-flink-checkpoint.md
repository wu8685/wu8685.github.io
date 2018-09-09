---
layout: post
title: Flink Checkpoint
categories: [flink]
tags: [flink]
---

在学习flink的时候看了本书《Stream Processing with Apache Flink》。里面对Flink checkpoint的原理讲得挺清楚的，后面内部分享时也参考了这个说法，所以这里按照我的理解描述一下。

首先，flink的checkpoint并不是将Subtask或者UDF对象进行序列化，然后保存。他们确实实现了Serializable接口，但是是为了要在Client，JobManager和TaskManager之间传输Graph。最终被checkpoint保存的每个subtask的状态只有raw state和managed state两种。raw state是用户自己进行序列化，而managed state是在operator生命周期初始化时就被注册到backend storage对象中了，在进行checkpoint时，会直接拿到注册的state进行保存（中间会调用回调函数，在UDF中对state进行赋值）。所以checkpoint的state不是很大的数据。

其次，checkpoint要保存的每个subtask的state并不是自然时刻下，他们的state。如下图所示，并不是要将Source1的3，Source2的4，Task1的2，Task2的3保存下来。如果要这样保存的话，那么on-the-fly的数据也要保存，否则无法从checkpoint中还原这个瞬间。checkpoint真正要保存的时刻是指的flink processing time或者event time概念下的瞬间。很显然，图中的这个瞬间至少Source和Task是不在一个“时刻”的，因为Source1的“时间”显然晚于Task1的“时间”。
![flink-checkpoint-01]({{ "/img/posts/2018-09-09-flink-checkpoint.md/flink-checkpoint-01.png" | prepend: site.baseurl }})

# Easy Understand
在理想情况下，checkpoint主要是完成一下这几个动作：

1. 暂停新数据的输入
2. 等待流中on-the-fly的数据被处理干净，此时得到flink graph的一个snapshot
3. 将所有Task中的State拷贝到State Backend中，如HDFS。此动作由各个Task Manager完成
4. 各个Task Manager将Task State的位置上报给Job Manager，完成checkpoint
5. 恢复数据的输入

如上所述，这里才需要“暂停输入+排干on-the-fly数据”的操作，这样才能拿到同一时刻下所有subtask的state。这又必然引入了STW的副作用。

# Chandy-Lamport algorithm

于是有了Chandy-Lamport算法。他解决的问题，就是在不停止流处理的前提下拿到每个subtask在某一瞬间的snapshot，从而完成checkpoint。

1. 在checkpoint触发时刻，Job Manager会往所有Source的流中放入一个barrier（图中三角形）。barrier包含当前checkpoint的ID
![flink-checkpoint-02]({{ "/img/posts/2018-09-09-flink-checkpoint.md/flink-checkpoint-02.png" | prepend: site.baseurl }})

2. 当barrier经过一个subtask时，即表示当前这个subtask处于checkpoint触发的“时刻”，他就会立即将barrier法往下游，并执行checkpoint方法将当前的state存入backend storage。图中Source1和Source2就是完成了checkpoint动作。
![flink-checkpoint-03]({{ "/img/posts/2018-09-09-flink-checkpoint.md/flink-checkpoint-03.png" | prepend: site.baseurl }})

3. 如果一个subtask有多个上游节点，这个subtask就需要等待所有上游发来的barrier都接收到，才能表示这个subtask到达了checkpoint触发“时刻”。但所有节点的barrier不一定一起到达，这时候就会面临“是否要对齐barrier”的问题（`Barrier Alignment`）。如图中的Task1.1，他有2个上游节点，Source1和Source2。假设Source1的barrier先到，这时候Task1.1就有2个选择：
	* 是马上把这个barrier发往下游并等待Source2的barrier来了再做checkpoint
	* 还是把Source1这边后续的event全都cache起来，等Source2的barrier来了，在做checkpoint，完了再继续处理Source1和Source2的event，当前Source1这边需要先处理cache里的event。

这引入了另一个概念：Result guarantees。

# Result Guarantees

Flink提供了几种容错机制：
* At-Most-Once
* At-Least-Once
* Exactly-Once
* End-to-end Exactly-Once

当不采用checkpoint时，每个event做多就只会被处理一次，这就是`At-Most-Once`。

当不开启Barrier对齐时，上图中的Source1来的在barrier后面的一些event有可能比Source2的barrier要先到Task1.1，因为我们没有cache这些event，所以他们会正常被处理并有可能更新Task1.1的state。这样，在回复checkpoint后，Task1.1的state可能就是处理了某些checkpoint“时刻”之后数据的状态。但是对于Source1来说，他还是会offset到正常的checkpoint“时刻”的位置，那么之前处理过的barrier后面的event可能还是会被再次放入这个流中。那么这些event就不能保证“只处理一次”了，有可能处理多次，这就是`At-Least-once`。

如果在Task1.1.处，先来的barrier后面的event都被cache了，那么就不会影响到这个task的state。那么Task1.1的checkpoint的state就能准确反映checkpoint“时刻”的情况。那么checkpoint回复后也不会有前面说的问题，这就是`Exactly-Once`。**但是因为Exactly-Once引入了cache机制，这会给checkpoint动作带来了额外的时延迟**。

`End-to-end Exactly-Once`需要结合外部系统一起完成，这里就不做讨论。