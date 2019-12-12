UTSDB-InfluxDB架构

UTSDB-InfluxDB采用存储与计算分离架构，由存储层+计算层构成，存储层基于Manul统一存储，简要框架如下：


通过UTSDB-InfluxDB写入的数据都将写入Manul统一存储中，Manul统一存储提供数据可靠性保证。如果物理机出现问题，调度系统可以随时在另一台机器上重启UTSDB，无需再做数据的迁移。大大降低运维成本，并提高服务可用性。
