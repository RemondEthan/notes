##### 1.GDE当前流程
![[GDE原有设计架构.png]]
~~~
数据流明细：
1.新开租户/已开通租户有材料主数据变化时，将材料主数据sink到Starrocks中，并发送MQ消息通知
2.新开租户/已开通租户有材料分类数据变化时，将材料分类数据sink到Starrocks中
2.Transform-Service通过监听MQ消息，并根据消息中数据反查Starrocks获取材料主数据与材料分类匹配数据，进行后续业务加工逻辑，完成后数据回写Starrocks
~~~
##### 2.工具链湖仓一体方案
![[GDE架构图.png]]
~~~
1.数据⼯具链通过dbz采集实时采集t_dic_item_material和t_dic_catalog两张表，并实现按租 户分区(按时间yyyymm分区待定) 
2.租户开通数据由GDE订阅，并进行持久化，用户租户变化消息通知时校验是否需要进行该租户下数据更新
3.数据⼯具链实现基于dbz cdc的消息通知，消息内容为表分区级别有更新，举例消息demo {"table": "t_dic_item_material", "partition": "tenant_id=tsxa1", "time": "20250529 154500"} 
4.消息通知由GDE服务进⾏订阅对应的topic，GDE服务取得消息后，主动从Starrocks查询数 据，查询条件tenant_id=${TENANT_ID} and time > ${last_query_time} and time <= ${NOW} 
5.GDE服务完成后，写⼊最新⼀次的时间上限，即${NOW},下次查询时作为 ${last_query_time}使⽤
~~~