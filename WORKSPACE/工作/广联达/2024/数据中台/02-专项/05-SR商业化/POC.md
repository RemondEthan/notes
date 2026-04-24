
###  1. Starrocks自3.2.8起，支持对节点进行打标签处理，可以在创建表或者物化视图是通过label来指定数据存放的位置。完成后，数据副本会在标签节点内平均分配，借此机制，可完成资源隔离功能。

### 2. 使用方式
~~~sql
ALTER SYSTEM MODIFY BACKEND "172.xx.xx.46:9050" SET ("labels.location" = "rack:rack1");
ALTER SYSTEM MODIFY BACKEND "172.xx.xx.47:9050" SET ("labels.location" = "rack:rack1");
ALTER SYSTEM MODIFY BACKEND "172.xx.xx.48:9050" SET ("labels.location" = "rack:rack2");
ALTER SYSTEM MODIFY BACKEND "172.xx.xx.49:9050" SET ("labels.location" = "rack:rack2");
ALTER SYSTEM MODIFY BACKEND "172.xx.xx.50:9050" SET ("labels.location" = "rack:rack3");
ALTER SYSTEM MODIFY BACKEND "172.xx.xx.51:9050" SET ("labels.location" = "rack:rack3");
~~~
###  3. 使用标签
  
  ~~~SQL
CREATE TABLE example_table (
    order_id bigint NOT NULL,
    dt date NOT NULL,
    user_id INT NOT NULL,
    good_id INT NOT NULL,
    cnt int NOT NULL,
    revenue int NOT NULL
)
PROPERTIES
("labels.location" = "rack:rack1,rack:rack2");
  ~~~
	  备注
	  如果您升级 StarRocks 至 3.2.8 或者以后版本，对于升级前已经创建的历史表，默认不使用标签分布数据。如果需要按照标签分布历史表数据，则可以执行如下语句，为历史表添加标签：
  ~~~sql
  ALTER TABLE example_table1  
  SET ("labels.location" = "rack:rack1,rack:rack2");
   ~~~

### 4. 使用标签指定物化视图的数据在 BE 节点上的分布
#### 建物化视图时[​](https://docs.starrocks.io/zh/docs/administration/management/resource_management/be_label/#%E5%BB%BA%E7%89%A9%E5%8C%96%E8%A7%86%E5%9B%BE%E6%97%B6 "建物化视图时的直接链接")

    对于新建的物化视图，属性 `labels.location` 默认为 `*` ，表示副本在所有标签中均匀分布。

	如果新建的物化视图的数据分布无需感知集群中服务器的地理位置信息，可以手动设置物化视图属性 `"labels.location" = ""`。
    建物化视图时指定物化视图的数据分布在 rack 1 和 rack 2，则可以执行如下语句：
    
~~~sql
CREATE MATERIALIZED VIEW mv_example_mv
DISTRIBUTED BY RANDOM
PROPERTIES (
"labels.location" = "rack:rack1,rack:rack2")
as 
select order_id, dt from example_table;
~~~


>当前进展？
>1.资源组目前测试从结果上来看，符合资源线控，需要补充sr日志
>2.跟进标签存档数据，目前MirrorShip反馈是bug，正在修改。

