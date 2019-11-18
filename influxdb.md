# InfluxDB专业术语

## database
对于users，retention policy，continuous query以及时序数据的一个逻辑上的集合。

## timestamp
数据点关联的日期和时间，在InfluxDB里的所有时间都是UTC的。

## field
InfluxDB数据中记录metadata和真实数据的键值对。fields在InfluxDB的数据结构中是必须的且不会被索引。如果要用field做查询条件的话，那就必须遍历所选时间范围里面的所有数据点，这种方式对比与tag效率会差很多。

## field key
组成field的键值对里面的键的部分。field key是字符串且保存在metadata中。

## field set
数据点上field key和field value的集合。

## field value
组成field的键值对里面的值的部分。field value才是真正的数据，可以是字符串，浮点数，整数，布尔型数据。一个field value总是和一个timestamp相关联。

field value不会被索引，如果要对field value做过滤话，那就必须遍历所选时间范围里面的所有数据点，这种方式对比与tag效率会差很多。

## measurement
InfluxDB数据结果中的一部分，描述了存在关联field中的数据的意义，measurement是字符串。

## point
InfluxDB数据结构的一部分由series中的的一堆field组成。 每个点由其series和timestamp唯一标识。

你不能在同一series中存储多个具有相同timestamp的点。 相反，当你使用与该series中现有点相同的timestamp记将新point写入同一series时，该field set将成为旧field set和新field set的并集。

## retention policy(RP)
InfluxDB数据结构的一部分，描述了InfluxDB保存数据的长短(duration)，数据存在集群里面的副本数(replication factor)，以及shard group的时间范围(shard group duration)。RPs在每个database里面是唯一的，连同measurement和tag set定义一个series。

当你创建一个database的时候，InfluxDB会自动创建一个叫做autogen的retention policy，其duration为永远，replication factor为1，shard group的duration设为的七天。


