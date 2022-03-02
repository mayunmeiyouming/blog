---
layout: post
title:  "Index Modules"
date:   2020-12-28 22:20:01 +0800
categories: [Tech]
tag: 
  - Index Modules
  - Elasticsearch
---

# Index Modules

索引模块是为每个索引创建的模块，用于控制与索引相关的所有方面。

## Index Settings

可以为每个索引设置索引级别

**static**

只能在索引创建时或在关闭的索引上设置。

**dynamic**

可以使用 update-index-settings API 来实时更新索引。 

`更改关闭的索引上的静态或动态索引设置可能会导致不正确的设置，如果不删除并重新创建索引，则无法纠正这些错误的设置。`

## Static index settings

以下是与任何特定索引模块都不相关的静态索引设置的列表：

`index.number_of_shards`

索引应具有的主要分片数。默认值为 1。只能在创建索引时设置，不能在关闭的索引上更改它。

每个索引的分片的最多为 1024 个 此限制是一个安全限制，可防止意外创建索引，该索引由于资源分配而可能破坏群集的稳定性。可以通过在群集的每个节点上指定 `export ES_JAVA_OPTS="-Des.index.max_number_of_shards=128"` 系统属性来修改此限制。

`index.number_of_routing_shards`

用于拆分索引的碎片的数量。

例如，将 `number_of_routing_shards` 设置为 30（5 x 2 x 3）可以以 2 或 3 的倍数进行分割。换句话说，可以按以下方式进行分割：

* 5 → 10 → 30 （依次为 2 和 3）

* 5 → 15 → 30 （依次为 3 和 2）

* 5 → 30（被 6 分割）

此设置的默认值取决于索引中的主分片数。默认值旨在允许您以 2 的因子进行拆分，最多可以分割 1024 个碎片。

`index.shard.check_on_startup`

打开前是否应检查碎片是否损坏。当检测到损坏时，它将防止分片被打开。接受：

`false` （默认）打开分片时不检查是否损坏。

`checksum` 检查物理损坏。

`true` 检查物理和逻辑损坏。就 CPU 和内存使用而言，这是非常昂贵的。

在大型索引上检查分片可能会花费很多时间。

`index.codec`

默认使用 LZ4 压缩算法来压缩存储的数据，但可以将其设置为 `best_compression`，即使用 `DEFLATE` 来获得更高的压缩比，但会降低存储字段的性能。如果要更新压缩类型，则合并段后将应用新的压缩类型。可以使用强制合并强制合并段。

`index.routing_partition_size`

自定义路由值可以到达的分片数量。默认为1，并且只能在创建索引时设置。除非 `index.number_of_shards` 值也为 1，否则该值必须小于 `index.number_of_shards`。有关如何使用此设置的更多详细信息，请参见路由到索引分区。

`index.soft_deletes.retention_lease.period`

在视为过期之前保留分片历史保留租约的最大期限。分片历史保留租约可确保在合并 Lucene 索引期间保留软删除。如果将软删除合并到一起，然后再将其复制到关注者，则由于领导者的历史记录不完整，因此以下过程将失败。默认为 12h。

`index.load_fixed_bitset_filters_eagerly`

指示是否为嵌套查询预加载缓存的过滤器。可能的值为 true（默认）和 false。

`index.hidden`

指示默认情况下是否应隐藏索引。使用通配符表达式时，默认情况下不返回隐藏索引。通过使用 `expand_wildcards` 参数，来控制每个请求的行为。值为 `true` 和 `false`（默认值）。

## Dynamic index settings

以下是与任何特定索引模块都不相关的所有动态索引设置的列表：

`index.number_of_replicas`

每个主分片具有的副本数。默认为 1。

`index.auto_expand_replicas`

根据集群中数据节点的数量自动扩展副本的数量。设置为以短划线分隔的上下限（例如 0 - 5）或将 all 用作上限（例如 0 - all）。 默认为 `false`（即已禁用）。请注意，副本的自动扩展数量仅考虑了分配过滤规则，而忽略了其他分配规则，例如分片分配意识和每个节点的总分片，如果适用的规则阻止了所有复制，这可能导致群集运行状况为黄色 复制副本被分配。

`index.search.idle.after`

分片在被视为搜索空闲之前无法接收搜索或获取请求的时间。（默认为30秒）

`index.refresh_interval`

执行刷新操作的频率，这使对索引的最近更改可见以进行搜索。默认为 1s。可以设置为 -1 以禁用刷新。 如果未明确设置此设置，则至少在 `index.search.idle.after` 秒之后仍未看到搜索流量的分片在收到搜索请求之前将不会接收后台刷新。命中空闲刷新的空闲分区的搜索将等待下一次后台刷新（在 1 秒内）。 此行为旨在在不执行搜索的情况下自动优化默认情况下的批量索引。为了退出此行为，应将显式值 1s 设置为刷新间隔。

`index.max_result_window`

搜索到此索引的最大值为+大小。默认值为 10000。搜索请求占用的堆内存和时间与 + 大小成正比，这将限制该内存。 有关提高此效果的更有效的选择，请参见滚动或之后搜索。

`index.max_inner_result_window`

内部匹配定义和顶部匹配聚合的最大+值到该索引。默认值为100。内部匹配和顶部匹配聚合占用的堆内存和时间与+大小成正比，因此会限制该内存。

`index.max_rescore_window`

搜索此索引时重新评分请求的 `window_size` 的最大值。默认为 `index.max_result_window`，默认为 10000。搜索请求占用的堆内存和时间与 `max`（window_size，from + size）成正比，因此会限制该内存。

`index.max_docvalue_fields_search`

查询中允许的最大 `docvalue_fields` 数。默认值为 100。Doc-Value字段很昂贵，因为它们可能会导致每个字段按文档搜索。

`index.max_script_fields`

查询中允许的最大script_fields数。默认为32

`index.max_ngram_diff`

`NGramTokenizer` 和 `NGramTokenFilter` 的 `min_gram` 和 `max_gram` 之间的最大允许差异。默认为 1。

`index.max_shingle_diff`

带状令牌过滤器的 `max_shingle_size` 和 `min_shingle_size` 之间允许的最大差异。默认为3。

`index.max_refresh_listeners`

索引的每个分片上可用的最大刷新侦听器数。这些侦听器用于实现 refresh = wait_for。

`index.analyze.max_token_count`

使用_analyze API可以产生的最大令牌数。默认为 10000

`index.highlight.max_analyzed_offset`

突出显示请求将分析的最大字符数。仅当在没有偏移量或术语向量的索引文本上要求突出显示时，此设置才适用。默认为1000000

`index.max_terms_count`

术语查询中可以使用的最大术语数。默认为 65536

`index.max_regex_length`

可在Regexp查询中使用的正则表达式的最大长度。默认为1000

索引路由分配启用

控制此索引的分片分配。可以设置为：

`all`（默认） - 允许为所有分片分配分片。

`primaries` - 仅允许为主要分片分配分片。

`new_primaries` - 仅允许为新创建的主分片分配分片。

`none` - 不允许分片分配。

索引路由重新平衡启用

为该索引启用分片重新平衡。 可以设置为：

全部（默认）-允许所有分片重新平衡。

`primaries` - 仅允许对主要碎片进行碎片重新平衡。

副本-仅允许副本分片重新平衡。

`none` - 不允许分片重新平衡。

`index.gc_deletes`

删除的文档的版本号可用于进一步版本控制的时间长度。默认值为60秒。

`index.default_pipeline`

此索引的默认摄取节点管道。如果设置了默认管道并且该管道不存在，则索引请求将失败。使用管道参数可以覆盖默认值。特殊管道名称_none表示不应运行任何摄取管道。

`index.final_pipeline`

此索引的最终摄取节点管道。如果设置了最终管道并且该管道不存在，则索引请求将失败。最终管道始终在请求管道（如果指定）和默认管道（如果存在）之后运行。特殊管道名称_none表示将不会运行任何摄取管道。
