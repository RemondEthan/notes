~~~sql
-- 元数据统计表设计--just设计
CREATE TABLE IF NOT EXISTS `metadata_object_statitics` (  
  `id` BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'auto increment id',  
  `object_id` INT UNSIGNED NOT NULL COMMENT 'Metadata object id',  
  `object_type` VARCHAR(128) NOT NULL COMMENT 'metadata object type',
  `statistic_name` VARCHAR(1024) NOT NULL COMMENT ``,
  `value` VARCHAR(20480),
  `system_stats` tiny(1),
  `modifitable` tiny(1),
  `auditInfo` VARCHAR(1024),  
  `deleted_at` BIGINT(20) UNSIGNED NOT NULL gitDEFAULT 0 COMMENT 'statistics deleted at',  
  PRIMARY KEY (`id`),  
  UNIQUE KEY `uk_oi_sn_da` (`object_id`, `statistics_name`, `deleted_at`),  
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT 'metadata object statistics';
~~~
- listener是如何进行加载的，时间传输到listener的机制是什么
- eventbus在这个过程中，时事件来源是什么
---
1. 如果使用gravitino做时间驱动，那我们需要用gravitino的rest catalog来做。