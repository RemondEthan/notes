~~~sql
CREATE VIEW `dws_pbp_wbs_quantity_expression_v` (
  `task_id` COMMENT "任务ID",
  `project_id` COMMENT "项目ID",
  `pbs_id` COMMENT "PBSID",
  `task_name` COMMENT "任务名称",
  `quantity_expression` COMMENT "工程量表达式",
  `element_type_id` COMMENT "构件类型ID",
  `coefficient` COMMENT "系数",
  `table_type` COMMENT "数据所属表编码",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  `dwd`.`t`.`id` AS `task_id`,
  `dwd`.`t`.`project_id`,
  CASE
    WHEN (`dwd`.`t`.`segment_id` IS NOT NULL) THEN `dwd`.`t`.`segment_id`
    ELSE `dwd`.`t`.`segment_group_id`
  END AS `pbs_id`,
  `dwd`.`t`.`task_name`,
  `dim`.`e`.`quantity_expression`,
  `dim`.`e`.`element_type_id`,
  `dim`.`e`.`coefficient`,
  `dwd`.`t`.`table_type`,
  `dim`.`e`.`system_type`
FROM
  `dim`.`dim_pbp_calculate_quantity_expression_v` AS `e`
  INNER JOIN `dwd`.`dwd_pbp_calculate_period_task_v` AS `t` ON (
    (`dwd`.`t`.`project_id` = `dim`.`e`.`project_id`)
    AND (`dwd`.`t`.`id` = `dim`.`e`.`task_id`)
  )
  AND (`dwd`.`t`.`table_type` = `dim`.`e`.`table_type`)
WHERE
  (`dwd`.`t`.`is_deleted` = 0)
  AND (`dim`.`e`.`is_deleted` = 0);
--
CREATE TABLE `gtc_f_element_type_material_quantity` (
  `project_id` bigint(20) NOT NULL COMMENT "项目ID",
  `phase_id` tinyint(4) NOT NULL COMMENT "阶段ID",
  `sub_special_id` tinyint(4) NOT NULL COMMENT "专业ID",
  `pf_building_id` largeint(40) NOT NULL COMMENT "单体ID",
  `pf_floor_id` largeint(40) NOT NULL COMMENT "楼层ID",
  `pf_construct_phase_id` largeint(40) NOT NULL COMMENT "施工阶段ID",
  `pf_construct_id` largeint(40) NOT NULL COMMENT "施工段ID",
  `region_id` largeint(40) NOT NULL COMMENT "区域ID",
  `floor_id` largeint(40) NOT NULL COMMENT "原始楼层ID",
  `construct_type_id` int(11) NOT NULL COMMENT "土建施工段类型ID",
  `construct_id` largeint(40) NOT NULL COMMENT "原始施工段ID",
  `element_type_id` int(11) NOT NULL COMMENT "构件类型",
  `material_id` largeint(40) NOT NULL COMMENT "材料ID",
  `channel_id` int(11) NOT NULL COMMENT "渠道ID",
  `system_type` varchar(32) NOT NULL DEFAULT "" COMMENT "系统类型",
  `pipe_type` tinyint(4) NOT NULL DEFAULT "-1" COMMENT "管道类型",
  `lmm_resource_code` varchar(255) REPLACE NULL DEFAULT "" COMMENT "编码",
  `material_name` varchar(255) REPLACE NULL DEFAULT "" COMMENT "材料名称",
  `specification` varchar(255) REPLACE NULL DEFAULT "" COMMENT "规格型号",
  `material_type_id` int(11) REPLACE NULL DEFAULT "0" COMMENT "材料类型ID",
  `material_type_name` varchar(255) REPLACE NULL DEFAULT " " COMMENT "材料类型名称",
  `element_type_name` varchar(255) REPLACE NULL DEFAULT " " COMMENT "构件类型名称",
  `qty_code` varchar(255) REPLACE NULL DEFAULT "" COMMENT "工程量表达式",
  `quantity` decimal(38, 6) REPLACE NULL DEFAULT "0.0" COMMENT "材料量",
  `unit` varchar(100) REPLACE NULL DEFAULT "KG" COMMENT "单位"
) ENGINE = OLAP AGGREGATE KEY(
  `project_id`,
  `phase_id`,
  `sub_special_id`,
  `pf_building_id`,
  `pf_floor_id`,
  `pf_construct_phase_id`,
  `pf_construct_id`,
  `region_id`,
  `floor_id`,
  `construct_type_id`,
  `construct_id`,
  `element_type_id`,
  `material_id`,
  `channel_id`,
  `system_type`,
  `pipe_type`
) COMMENT "构件类型材料量明细表" DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 PROPERTIES (
  "replication_num" = "3",
  "in_memory" = "false",
  "storage_format" = "DEFAULT",
  "enable_persistent_index" = "true",
  "compression" = "LZ4"
);
--
CREATE VIEW `dwd_pbp_calculate_element_type_material_quantity_v` (
  `project_id` COMMENT "项目ID",
  `sub_special_id` COMMENT "专业ID",
  `region_id` COMMENT "区域ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `construction_phase_id` COMMENT "施工阶段ID",
  `segment_id` COMMENT "施工段ID",
  `original_floor_id` COMMENT "原始楼层ID",
  `original_construction_phase_id` COMMENT "原始施工阶段ID",
  `original_segment_id` COMMENT "原始施工段ID",
  `element_type_id` COMMENT "构件类型",
  `element_type_name` COMMENT "构件类型名称",
  `material_id` COMMENT "材料ID",
  `material_code` COMMENT "材料编码",
  `material_name` COMMENT "材料名称",
  `material_specification` COMMENT "材料规格型号",
  `material_type_id` COMMENT "材料类型ID",
  `material_type_name` COMMENT "材料类型名称",
  `quantity_type_code` COMMENT "算量类型编码",
  `quantity_expression` COMMENT "算量表达式",
  `quantity_value` COMMENT "算量值",
  `measure_unit` COMMENT "算量单位",
  `channel_id` COMMENT "渠道ID",
  `pipe_type` COMMENT "管道类型，默认值-1，-1表示无；1表示直管；2表示立管",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  `gtcp_quantity`.`q`.`project_id`,
  `gtcp_quantity`.`q`.`sub_special_id`,
  `gtcp_quantity`.`q`.`region_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_building_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_building_id`
  END AS `building_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_floor_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_floor_id`
  END AS `floor_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_construct_phase_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_construct_phase_id`
  END AS `construction_phase_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_construct_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_construct_id`
  END AS `segment_id`,
  `gtcp_quantity`.`q`.`floor_id` AS `original_floor_id`,
  `gtcp_quantity`.`q`.`construct_type_id` AS `original_construction_phase_id`,
  `gtcp_quantity`.`q`.`construct_id` AS `original_segment_id`,
  `gtcp_quantity`.`q`.`element_type_id`,
  `gtcp_quantity`.`q`.`element_type_name`,
  `gtcp_quantity`.`q`.`material_id`,
  `gtcp_quantity`.`q`.`lmm_resource_code` AS `material_code`,
  `gtcp_quantity`.`q`.`material_name`,
  `gtcp_quantity`.`q`.`specification` AS `material_specification`,
  `gtcp_quantity`.`q`.`material_type_id`,
  `gtcp_quantity`.`q`.`material_type_name`,
  `gtcp_quantity`.`q`.`phase_id` AS `quantity_type_code`,
  `gtcp_quantity`.`q`.`qty_code` AS `quantity_expression`,
  `gtcp_quantity`.`q`.`quantity` AS `quantity_value`,
  `gtcp_quantity`.`q`.`unit` AS `measure_unit`,
  `gtcp_quantity`.`q`.`channel_id`,
  `gtcp_quantity`.`q`.`pipe_type`,
  CASE
    WHEN ((length(`gtcp_quantity`.`q`.`system_type`)) = 0) THEN -1
    WHEN (`gtcp_quantity`.`q`.`system_type` IS NULL) THEN -1
    ELSE `gtcp_quantity`.`q`.`system_type`
  END AS `system_type`
FROM
  `gtcp_quantity`.`gtc_f_element_type_material_quantity` AS `q`;
--
CREATE VIEW `dwd_pbp_calculate_element_type_material_quantity_v` (
  `project_id` COMMENT "项目ID",
  `sub_special_id` COMMENT "专业ID",
  `region_id` COMMENT "区域ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `construction_phase_id` COMMENT "施工阶段ID",
  `segment_id` COMMENT "施工段ID",
  `original_floor_id` COMMENT "原始楼层ID",
  `original_construction_phase_id` COMMENT "原始施工阶段ID",
  `original_segment_id` COMMENT "原始施工段ID",
  `element_type_id` COMMENT "构件类型",
  `element_type_name` COMMENT "构件类型名称",
  `material_id` COMMENT "材料ID",
  `material_code` COMMENT "材料编码",
  `material_name` COMMENT "材料名称",
  `material_specification` COMMENT "材料规格型号",
  `material_type_id` COMMENT "材料类型ID",
  `material_type_name` COMMENT "材料类型名称",
  `quantity_type_code` COMMENT "算量类型编码",
  `quantity_expression` COMMENT "算量表达式",
  `quantity_value` COMMENT "算量值",
  `measure_unit` COMMENT "算量单位",
  `channel_id` COMMENT "渠道ID",
  `pipe_type` COMMENT "管道类型，默认值-1，-1表示无；1表示直管；2表示立管",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  `gtcp_quantity`.`q`.`project_id`,
  `gtcp_quantity`.`q`.`sub_special_id`,
  `gtcp_quantity`.`q`.`region_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_building_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_building_id`
  END AS `building_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_floor_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_floor_id`
  END AS `floor_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_construct_phase_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_construct_phase_id`
  END AS `construction_phase_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_construct_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_construct_id`
  END AS `segment_id`,
  `gtcp_quantity`.`q`.`floor_id` AS `original_floor_id`,
  `gtcp_quantity`.`q`.`construct_type_id` AS `original_construction_phase_id`,
  `gtcp_quantity`.`q`.`construct_id` AS `original_segment_id`,
  `gtcp_quantity`.`q`.`element_type_id`,
  `gtcp_quantity`.`q`.`element_type_name`,
  `gtcp_quantity`.`q`.`material_id`,
  `gtcp_quantity`.`q`.`lmm_resource_code` AS `material_code`,
  `gtcp_quantity`.`q`.`material_name`,
  `gtcp_quantity`.`q`.`specification` AS `material_specification`,
  `gtcp_quantity`.`q`.`material_type_id`,
  `gtcp_quantity`.`q`.`material_type_name`,
  `gtcp_quantity`.`q`.`phase_id` AS `quantity_type_code`,
  `gtcp_quantity`.`q`.`qty_code` AS `quantity_expression`,
  `gtcp_quantity`.`q`.`quantity` AS `quantity_value`,
  `gtcp_quantity`.`q`.`unit` AS `measure_unit`,
  `gtcp_quantity`.`q`.`channel_id`,
  `gtcp_quantity`.`q`.`pipe_type`,
  CASE
    WHEN ((length(`gtcp_quantity`.`q`.`system_type`)) = 0) THEN -1
    WHEN (`gtcp_quantity`.`q`.`system_type` IS NULL) THEN -1
    ELSE `gtcp_quantity`.`q`.`system_type`
  END AS `system_type`
FROM
  `gtcp_quantity`.`gtc_f_element_type_material_quantity` AS `q`;
--
CREATE VIEW `dws_pbp_pbs_sub_item_material_quantity_detail_v` (
  `pbs_id` COMMENT "PBSID",
  `full_id` COMMENT "全路径ID",
  `project_id` COMMENT "项目ID",
  `sub_special_id` COMMENT "专业ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `construction_phase_id` COMMENT "施工阶段ID",
  `segment_id` COMMENT "施工段ID",
  `element_type_id` COMMENT "构件类型ID",
  `element_type_name` COMMENT "构件类型名称",
  `material_id` COMMENT "材料ID",
  `material_code` COMMENT "材料编码",
  `material_name` COMMENT "材料名称",
  `material_specification` COMMENT "材料规格",
  `material_type_id` COMMENT "材料类型ID",
  `material_type_name` COMMENT "材料类型名称",
  `quantity_type_code` COMMENT "算量类别编码",
  `quantity_expression` COMMENT "算量表达式",
  `quantity_value` COMMENT "算量数值",
  `measure_unit` COMMENT "算量单位",
  `channel_id` COMMENT "渠道ID",
  `sub_item_id` COMMENT "分部/子分部ID",
  `structure_type_name` COMMENT "所属部位类型",
  `system_type` COMMENT "系统类型",
  `pipe_type` COMMENT "管道类型",
  `table_type` COMMENT "数据所属表编码"
) AS
SELECT
  `total`.`pbs_id`,
  `total`.`full_id`,
  `total`.`project_id`,
  `total`.`sub_special_id`,
  `total`.`building_id`,
  `total`.`floor_id`,
  `total`.`construction_phase_id`,
  `total`.`segment_id`,
  `total`.`element_type_id`,
  `total`.`element_type_name`,
  `total`.`material_id`,
  `total`.`material_code`,
  `total`.`material_name`,
  `total`.`material_specification`,
  `total`.`material_type_id`,
  `total`.`material_type_name`,
  `total`.`quantity_type_code`,
  `total`.`quantity_expression`,
  `total`.`quantity_value`,
  `total`.`measure_unit`,
  `total`.`channel_id`,
  `total`.`sub_item_id`,
  `total`.`structure_type_name`,
  `total`.`system_type`,
  `total`.`pipe_type`,
  `total`.`table_type`
FROM
  (
    SELECT
      `dwd`.`p`.`id` AS `pbs_id`,
      `dwd`.`p`.`full_id`,
      `dwd`.`q`.`project_id`,
      `dwd`.`q`.`sub_special_id`,
      `dwd`.`q`.`building_id`,
      `dwd`.`q`.`floor_id`,
      `dwd`.`q`.`construction_phase_id`,
      `dwd`.`q`.`segment_id`,
      `dwd`.`q`.`element_type_id`,
      `dwd`.`q`.`element_type_name`,
      `dwd`.`q`.`material_id`,
      `dwd`.`q`.`material_code`,
      `dwd`.`q`.`material_name`,
      `dwd`.`q`.`material_specification`,
      `dwd`.`q`.`material_type_id`,
      `dwd`.`q`.`material_type_name`,
      `dwd`.`q`.`quantity_type_code`,
      `dwd`.`q`.`quantity_expression`,
      `dwd`.`q`.`quantity_value`,
      `dwd`.`q`.`measure_unit`,
      `dwd`.`q`.`channel_id`,
      `dwd`.`p`.`sub_item_id`,
      `dwd`.`q`.`region_id`,
      'other' AS `structure_type_name`,
      CASE
        WHEN ((length(`dwd`.`q`.`system_type`)) = 0) THEN -1
        WHEN (`dwd`.`q`.`system_type` IS NULL) THEN -1
        ELSE `dwd`.`q`.`system_type`
      END AS `system_type`,
      `dwd`.`q`.`pipe_type`,
      `dwd`.`p`.`table_type`
    FROM
      `dwd`.`dwd_pbp_calculate_element_type_material_quantity_v` AS `q`
      INNER JOIN `dwd`.`dwd_pbp_pbs_structure_sub_item_fill_v` AS `p` ON (
        (`dwd`.`p`.`project_id` = `dwd`.`q`.`project_id`)
        AND (
          `dwd`.`p`.`building_id` = `dwd`.`q`.`building_id`
        )
      )
      AND (`dwd`.`p`.`floor_id` = `dwd`.`q`.`floor_id`)
    WHERE
      `dwd`.`p`.`is_deleted` = 0
    UNION ALL
    SELECT
      `dwd`.`p`.`id` AS `pbs_id`,
      `dwd`.`p`.`full_id`,
      `dwd`.`q`.`project_id`,
      `dwd`.`q`.`sub_special_id`,
      `dwd`.`q`.`building_id`,
      `dwd`.`q`.`floor_id`,
      `dwd`.`q`.`construction_phase_id`,
      `dwd`.`q`.`segment_id`,
      `dwd`.`q`.`element_type_id`,
      `dwd`.`q`.`element_type_name`,
      `dwd`.`q`.`material_id`,
      `dwd`.`q`.`material_code`,
      `dwd`.`q`.`material_name`,
      `dwd`.`q`.`material_specification`,
      `dwd`.`q`.`material_type_id`,
      `dwd`.`q`.`material_type_name`,
      `dwd`.`q`.`quantity_type_code`,
      `dwd`.`q`.`quantity_expression`,
      `dwd`.`q`.`quantity_value`,
      `dwd`.`q`.`measure_unit`,
      `dwd`.`q`.`channel_id`,
      `dwd`.`p`.`sub_item_id`,
      `dwd`.`q`.`region_id`,
      'building' AS `structure_type_name`,
      CASE
        WHEN ((length(`dwd`.`q`.`system_type`)) = 0) THEN -1
        WHEN (`dwd`.`q`.`system_type` IS NULL) THEN -1
        ELSE `dwd`.`q`.`system_type`
      END AS `system_type`,
      `dwd`.`q`.`pipe_type`,
      `dwd`.`p`.`table_type`
    FROM
      `dwd`.`dwd_pbp_calculate_element_type_material_quantity_v` AS `q`
      INNER JOIN `dwd`.`dwd_pbp_pbs_structure_sub_item_fill_v` AS `p` ON (`dwd`.`p`.`project_id` = `dwd`.`q`.`project_id`)
      AND (
        `dwd`.`p`.`building_id` = `dwd`.`q`.`building_id`
      )
    WHERE
      `dwd`.`p`.`is_deleted` = 0
  ) total;
--
CREATE VIEW `dws_pbp_wbs_sub_item_material_quantity_detail_v` (
  `project_id` COMMENT "项目ID",
  `full_id` COMMENT "全路径ID",
  `pbs_id` COMMENT "PBSID",
  `task_id` COMMENT "任务ID",
  `element_type_id` COMMENT "构件类型ID",
  `element_type_name` COMMENT "构件类型名称",
  `quantity_type_code` COMMENT "算量类别编码",
  `material_code` COMMENT "材料编码",
  `material_id` COMMENT "材料ID",
  `material_name` COMMENT "材料名称",
  `material_type_id` COMMENT "材料类型ID",
  `material_type_name` COMMENT "材料类型名称",
  `material_specification` COMMENT "材料规格型号",
  `quantity_expression` COMMENT "算量表达式",
  `measure_unit` COMMENT "算量单位",
  `quantity_value` COMMENT "算量值",
  `channel_id` COMMENT "渠道ID",
  `sub_item_id` COMMENT "分部/子分部ID",
  `structure_type_name` COMMENT "所属部位类型",
  `system_type` COMMENT "系统类型",
  `pipe_type` COMMENT "管道类型",
  `table_type` COMMENT "数据所属表编码"
) AS
SELECT
  `dws`.`q`.`project_id`,
  `dws`.`q`.`full_id`,
  `dws`.`q`.`pbs_id`,
  `dws`.`e`.`task_id`,
  `dws`.`q`.`element_type_id`,
  `dws`.`q`.`element_type_name`,
  `dws`.`q`.`quantity_type_code`,
  `dws`.`q`.`material_code`,
  `dws`.`q`.`material_id`,
  `dws`.`q`.`material_name`,
  `dws`.`q`.`material_type_id`,
  `dws`.`q`.`material_type_name`,
  `dws`.`q`.`material_specification`,
  `dws`.`q`.`quantity_expression`,
  `dws`.`q`.`measure_unit`,
  `dws`.`q`.`quantity_value` * `dws`.`e`.`coefficient` AS `quantity_value`,
  `dws`.`q`.`channel_id`,
  `dws`.`q`.`sub_item_id`,
  `dws`.`q`.`structure_type_name`,
  `dws`.`q`.`system_type`,
  `dws`.`q`.`pipe_type`,
  `dws`.`q`.`table_type`
FROM
  `dws`.`dws_pbp_pbs_sub_item_material_quantity_detail_v` AS `q`
  INNER JOIN `dws`.`dws_pbp_wbs_quantity_expression_v` AS `e` ON (
    (
      (
        (
          (`dws`.`q`.`project_id` = `dws`.`e`.`project_id`)
          AND (`dws`.`q`.`pbs_id` = `dws`.`e`.`pbs_id`)
        )
        AND (
          `dws`.`q`.`quantity_expression` = `dws`.`e`.`quantity_expression`
        )
      )
      AND (
        `dws`.`q`.`element_type_id` = `dws`.`e`.`element_type_id`
      )
    )
    AND (`dws`.`q`.`table_type` = `dws`.`e`.`table_type`)
  )
  AND (
    `dws`.`q`.`system_type` = `dws`.`e`.`system_type`
  );
--
CREATE VIEW `dim_pbp_calculate_quantity_expression_v` (
  `id` COMMENT "主键ID",
  `project_id` COMMENT "项目ID",
  `task_id` COMMENT "任务ID",
  `quantity_expression` COMMENT "工程量表达式",
  `element_type_id` COMMENT "构件类型ID",
  `coefficient` COMMENT "系数",
  `is_deleted` COMMENT "是否删除",
  `sys_revision` COMMENT "系统-乐观锁版本号",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modifier_id` COMMENT "系统-最后修改人",
  `table_type` COMMENT "数据所属表编码",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_modifier_id`,
  '0_bim_202005181910_0' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_modifier_id`,
  '1_bim_202005181910_0' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_modifier_id`,
  '2_bim_202005181910_1' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_modifier_id`,
  '3_bim_202005181910_1' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_modifier_id`,
  '0_bim_1120655100311625730_0' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_modifier_id`,
  '1_bim_1120655100311625730_0' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_modifier_id`,
  '2_bim_1120655100311625730_1' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_modifier_id`,
  '3_bim_1120655100311625730_1' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`;
--
CREATE VIEW `dws_pbp_wbs_quantity_expression_v` (
  `task_id` COMMENT "任务ID",
  `project_id` COMMENT "项目ID",
  `pbs_id` COMMENT "PBSID",
  `task_name` COMMENT "任务名称",
  `quantity_expression` COMMENT "工程量表达式",
  `element_type_id` COMMENT "构件类型ID",
  `coefficient` COMMENT "系数",
  `table_type` COMMENT "数据所属表编码",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  `dwd`.`t`.`id` AS `task_id`,
  `dwd`.`t`.`project_id`,
  CASE
    WHEN (`dwd`.`t`.`segment_id` IS NOT NULL) THEN `dwd`.`t`.`segment_id`
    ELSE `dwd`.`t`.`segment_group_id`
  END AS `pbs_id`,
  `dwd`.`t`.`task_name`,
  `dim`.`e`.`quantity_expression`,
  `dim`.`e`.`element_type_id`,
  `dim`.`e`.`coefficient`,
  `dwd`.`t`.`table_type`,
  `dim`.`e`.`system_type`
FROM
  `dim`.`dim_pbp_calculate_quantity_expression_v` AS `e`
  INNER JOIN `dwd`.`dwd_pbp_calculate_period_task_v` AS `t` ON (
    (`dwd`.`t`.`project_id` = `dim`.`e`.`project_id`)
    AND (`dwd`.`t`.`id` = `dim`.`e`.`task_id`)
  )
  AND (`dwd`.`t`.`table_type` = `dim`.`e`.`table_type`)
WHERE
  (`dwd`.`t`.`is_deleted` = 0)
  AND (`dim`.`e`.`is_deleted` = 0);
--
CREATE VIEW `dws_pbp_wbs_material_quantity_detail_v` (
  `project_id` COMMENT "项目ID",
  `full_id` COMMENT "全路径ID",
  `pbs_id` COMMENT "PBSID",
  `task_id` COMMENT "任务ID",
  `element_type_id` COMMENT "构件类型ID",
  `element_type_name` COMMENT "构件类型名称",
  `quantity_type_code` COMMENT "算量类别编码",
  `material_code` COMMENT "材料编码",
  `material_id` COMMENT "材料ID",
  `material_name` COMMENT "材料名称",
  `material_type_id` COMMENT "材料类型ID",
  `material_type_name` COMMENT "材料类型名称",
  `material_specification` COMMENT "材料规格型号",
  `quantity_expression` COMMENT "算量表达式",
  `measure_unit` COMMENT "算量单位",
  `quantity_value` COMMENT "算量值",
  `channel_id` COMMENT "渠道ID",
  `system_type` COMMENT "数据所属表编码",
  `pipe_type` COMMENT "管道类型，默认值-1，-1表示无；1表示直管，2表示立管",
  `table_type` COMMENT "系统类型"
) AS
SELECT
  `dws`.`q`.`project_id`,
  `dws`.`q`.`full_id`,
  `dws`.`q`.`pbs_id`,
  `dws`.`e`.`task_id`,
  `dws`.`q`.`element_type_id`,
  `dws`.`q`.`element_type_name`,
  `dws`.`q`.`quantity_type_code`,
  `dws`.`q`.`material_code`,
  `dws`.`q`.`material_id`,
  `dws`.`q`.`material_name`,
  `dws`.`q`.`material_type_id`,
  `dws`.`q`.`material_type_name`,
  `dws`.`q`.`material_specification`,
  `dws`.`q`.`quantity_expression`,
  `dws`.`q`.`measure_unit`,
  `dws`.`q`.`quantity_value` * `dws`.`e`.`coefficient` AS `quantity_value`,
  `dws`.`q`.`channel_id`,
  `dws`.`q`.`system_type`,
  `dws`.`q`.`pipe_type`,
  `dws`.`q`.`table_type`
FROM
  `dws`.`dws_pbp_pbs_material_quantity_detail_v` AS `q`
  INNER JOIN `dws`.`dws_pbp_wbs_quantity_expression_v` AS `e` ON (
    (
      (
        (
          (`dws`.`q`.`project_id` = `dws`.`e`.`project_id`)
          AND (`dws`.`q`.`pbs_id` = `dws`.`e`.`pbs_id`)
        )
        AND (
          `dws`.`q`.`quantity_expression` = `dws`.`e`.`quantity_expression`
        )
      )
      AND (
        `dws`.`q`.`element_type_id` = `dws`.`e`.`element_type_id`
      )
    )
    AND (`dws`.`q`.`table_type` = `dws`.`e`.`table_type`)
  )
  AND (
    `dws`.`q`.`system_type` = `dws`.`e`.`system_type`
  );
--
CREATE VIEW `dws_pbp_pbs_material_quantity_detail_v` (
  `pbs_id` COMMENT "PBSID",
  `full_id` COMMENT "全路径ID",
  `project_id` COMMENT "项目ID",
  `sub_special_id` COMMENT "专业ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `construction_phase_id` COMMENT "施工阶段ID",
  `segment_id` COMMENT "施工段ID",
  `element_type_id` COMMENT "构件类型ID",
  `element_type_name` COMMENT "构件类型名称",
  `material_id` COMMENT "材料ID",
  `material_code` COMMENT "材料编码",
  `material_name` COMMENT "材料名称",
  `material_specification` COMMENT "材料规格",
  `material_type_id` COMMENT "材料类型ID",
  `material_type_name` COMMENT "材料类型名称",
  `quantity_type_code` COMMENT "算量类别编码",
  `quantity_expression` COMMENT "算量表达式",
  `quantity_value` COMMENT "算量数值",
  `measure_unit` COMMENT "算量单位",
  `channel_id` COMMENT "渠道ID",
  `system_type` COMMENT "系统类型",
  `pipe_type` COMMENT "管道类型，默认值-1，-1表示无；1表示直管，2表示立管",
  `table_type` COMMENT "数据所属表编码"
) AS
SELECT
  `dwd`.`p`.`id` AS `pbs_id`,
  `dwd`.`p`.`full_id`,
  `dwd`.`q`.`project_id`,
  `dwd`.`q`.`sub_special_id`,
  `dwd`.`q`.`building_id`,
  `dwd`.`q`.`floor_id`,
  `dwd`.`q`.`construction_phase_id`,
  `dwd`.`q`.`segment_id`,
  `dwd`.`q`.`element_type_id`,
  `dwd`.`q`.`element_type_name`,
  `dwd`.`q`.`material_id`,
  `dwd`.`q`.`material_code`,
  `dwd`.`q`.`material_name`,
  `dwd`.`q`.`material_specification`,
  `dwd`.`q`.`material_type_id`,
  `dwd`.`q`.`material_type_name`,
  `dwd`.`q`.`quantity_type_code`,
  `dwd`.`q`.`quantity_expression`,
  `dwd`.`q`.`quantity_value`,
  `dwd`.`q`.`measure_unit`,
  `dwd`.`q`.`channel_id`,
  `dwd`.`q`.`system_type`,
  `dwd`.`q`.`pipe_type`,
  `dwd`.`p`.`table_type`
FROM
  `dwd`.`dwd_pbp_pbs_structure_fill_v` AS `p`
  INNER JOIN `dwd`.`dwd_pbp_calculate_element_type_material_quantity_v` AS `q` ON (
    (
      (
        (`dwd`.`p`.`project_id` = `dwd`.`q`.`project_id`)
        AND (
          `dwd`.`p`.`building_id` = `dwd`.`q`.`building_id`
        )
      )
      AND (`dwd`.`p`.`floor_id` = `dwd`.`q`.`floor_id`)
    )
    AND (
      `dwd`.`p`.`construction_phase_id` = `dwd`.`q`.`construction_phase_id`
    )
  )
  AND (
    `dwd`.`p`.`model_segment_id` = `dwd`.`q`.`segment_id`
  )
WHERE
  `dwd`.`p`.`is_deleted` = 0;
--
CREATE VIEW `dws_pbp_pbs_material_quantity_detail_v` (
  `pbs_id` COMMENT "PBSID",
  `full_id` COMMENT "全路径ID",
  `project_id` COMMENT "项目ID",
  `sub_special_id` COMMENT "专业ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `construction_phase_id` COMMENT "施工阶段ID",
  `segment_id` COMMENT "施工段ID",
  `element_type_id` COMMENT "构件类型ID",
  `element_type_name` COMMENT "构件类型名称",
  `material_id` COMMENT "材料ID",
  `material_code` COMMENT "材料编码",
  `material_name` COMMENT "材料名称",
  `material_specification` COMMENT "材料规格",
  `material_type_id` COMMENT "材料类型ID",
  `material_type_name` COMMENT "材料类型名称",
  `quantity_type_code` COMMENT "算量类别编码",
  `quantity_expression` COMMENT "算量表达式",
  `quantity_value` COMMENT "算量数值",
  `measure_unit` COMMENT "算量单位",
  `channel_id` COMMENT "渠道ID",
  `system_type` COMMENT "系统类型",
  `pipe_type` COMMENT "管道类型，默认值-1，-1表示无；1表示直管，2表示立管",
  `table_type` COMMENT "数据所属表编码"
) AS
SELECT
  `dwd`.`p`.`id` AS `pbs_id`,
  `dwd`.`p`.`full_id`,
  `dwd`.`q`.`project_id`,
  `dwd`.`q`.`sub_special_id`,
  `dwd`.`q`.`building_id`,
  `dwd`.`q`.`floor_id`,
  `dwd`.`q`.`construction_phase_id`,
  `dwd`.`q`.`segment_id`,
  `dwd`.`q`.`element_type_id`,
  `dwd`.`q`.`element_type_name`,
  `dwd`.`q`.`material_id`,
  `dwd`.`q`.`material_code`,
  `dwd`.`q`.`material_name`,
  `dwd`.`q`.`material_specification`,
  `dwd`.`q`.`material_type_id`,
  `dwd`.`q`.`material_type_name`,
  `dwd`.`q`.`quantity_type_code`,
  `dwd`.`q`.`quantity_expression`,
  `dwd`.`q`.`quantity_value`,
  `dwd`.`q`.`measure_unit`,
  `dwd`.`q`.`channel_id`,
  `dwd`.`q`.`system_type`,
  `dwd`.`q`.`pipe_type`,
  `dwd`.`p`.`table_type`
FROM
  `dwd`.`dwd_pbp_pbs_structure_fill_v` AS `p`
  INNER JOIN `dwd`.`dwd_pbp_calculate_element_type_material_quantity_v` AS `q` ON (
    (
      (
        (`dwd`.`p`.`project_id` = `dwd`.`q`.`project_id`)
        AND (
          `dwd`.`p`.`building_id` = `dwd`.`q`.`building_id`
        )
      )
      AND (`dwd`.`p`.`floor_id` = `dwd`.`q`.`floor_id`)
    )
    AND (
      `dwd`.`p`.`construction_phase_id` = `dwd`.`q`.`construction_phase_id`
    )
  )
  AND (
    `dwd`.`p`.`model_segment_id` = `dwd`.`q`.`segment_id`
  )
WHERE
  `dwd`.`p`.`is_deleted` = 0;
--
CREATE VIEW `dim_pbp_calculate_quantity_expression_v` (
  `id` COMMENT "主键ID",
  `project_id` COMMENT "项目ID",
  `task_id` COMMENT "任务ID",
  `quantity_expression` COMMENT "工程量表达式",
  `element_type_id` COMMENT "构件类型ID",
  `coefficient` COMMENT "系数",
  `is_deleted` COMMENT "是否删除",
  `sys_revision` COMMENT "系统-乐观锁版本号",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modifier_id` COMMENT "系统-最后修改人",
  `table_type` COMMENT "数据所属表编码",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`sys_modifier_id`,
  '0_bim_202005181910_0' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`sys_modifier_id`,
  '1_bim_202005181910_0' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_1_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`sys_modifier_id`,
  '2_bim_202005181910_1' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_2_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`sys_modifier_id`,
  '3_bim_202005181910_1' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_3_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`sys_modifier_id`,
  '0_bim_1120655100311625730_0' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_0_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`sys_modifier_id`,
  '1_bim_1120655100311625730_0' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_1_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`sys_modifier_id`,
  '2_bim_1120655100311625730_1' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_2_bim_30_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`relate_task_id` AS `task_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`quantity_expression`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`element_type_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`coefficient`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_revision`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`sys_modifier_id`,
  '3_bim_1120655100311625730_1' AS `table_type`,
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`.`system_type`
FROM
  `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`;
--
CREATE VIEW `dws_pbp_wbs_quantity_expression_v` (
  `task_id` COMMENT "任务ID",
  `project_id` COMMENT "项目ID",
  `pbs_id` COMMENT "PBSID",
  `task_name` COMMENT "任务名称",
  `quantity_expression` COMMENT "工程量表达式",
  `element_type_id` COMMENT "构件类型ID",
  `coefficient` COMMENT "系数",
  `table_type` COMMENT "数据所属表编码",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  `dwd`.`t`.`id` AS `task_id`,
  `dwd`.`t`.`project_id`,
  CASE
    WHEN (`dwd`.`t`.`segment_id` IS NOT NULL) THEN `dwd`.`t`.`segment_id`
    ELSE `dwd`.`t`.`segment_group_id`
  END AS `pbs_id`,
  `dwd`.`t`.`task_name`,
  `dim`.`e`.`quantity_expression`,
  `dim`.`e`.`element_type_id`,
  `dim`.`e`.`coefficient`,
  `dwd`.`t`.`table_type`,
  `dim`.`e`.`system_type`
FROM
  `dim`.`dim_pbp_calculate_quantity_expression_v` AS `e`
  INNER JOIN `dwd`.`dwd_pbp_calculate_period_task_v` AS `t` ON (
    (`dwd`.`t`.`project_id` = `dim`.`e`.`project_id`)
    AND (`dwd`.`t`.`id` = `dim`.`e`.`task_id`)
  )
  AND (`dwd`.`t`.`table_type` = `dim`.`e`.`table_type`)
WHERE
  (`dwd`.`t`.`is_deleted` = 0)
  AND (`dim`.`e`.`is_deleted` = 0);
--
CREATE VIEW `dws_pbp_wbs_model_quantity_detail_v` (
  `project_id` COMMENT "项目ID",
  `table_type` COMMENT "数据所属表编码",
  `full_id` COMMENT "全路径ID",
  `pbs_id` COMMENT "PBSID",
  `task_id` COMMENT "任务ID",
  `element_id` COMMENT "构件ID",
  `element_name` COMMENT "构件名称",
  `element_type_id` COMMENT "构件类型ID",
  `element_type_name` COMMENT "构件类型名称",
  `quantity_type_code` COMMENT "算量类别编码",
  `quantity_expression` COMMENT "算量表达式",
  `quantity_name` COMMENT "算量名称",
  `measure_unit` COMMENT "算量单位",
  `quantity_value` COMMENT "算量值",
  `model_specification` COMMENT "模型规格"
) AS
SELECT
  `dws`.`q`.`project_id`,
  `dws`.`q`.`table_type`,
  `dws`.`q`.`full_id`,
  `dws`.`q`.`pbs_id`,
  `dws`.`e`.`task_id`,
  `dws`.`q`.`element_id`,
  `dws`.`q`.`element_name`,
  `dws`.`q`.`element_type_id`,
  `dws`.`q`.`element_type_name`,
  `dws`.`q`.`quantity_type_code`,
  `dws`.`e`.`quantity_expression`,
  `dws`.`q`.`quantity_name`,
  `dws`.`q`.`measure_unit`,
  `dws`.`q`.`quantity_value` * `dws`.`e`.`coefficient` AS `quantity_value`,
  `dws`.`q`.`model_specification`
FROM
  `dws`.`dws_pbp_pbs_model_quantity_detail_v` AS `q`
  INNER JOIN `dws`.`dws_pbp_wbs_quantity_expression_v` AS `e` ON (
    (
      (
        (`dws`.`q`.`project_id` = `dws`.`e`.`project_id`)
        AND (`dws`.`q`.`table_type` = `dws`.`e`.`table_type`)
      )
      AND (`dws`.`q`.`pbs_id` = `dws`.`e`.`pbs_id`)
    )
    AND (
      `dws`.`q`.`quantity_expression` = `dws`.`e`.`quantity_expression`
    )
  )
  AND (
    `dws`.`q`.`element_type_id` = `dws`.`e`.`element_type_id`
  );
--
CREATE VIEW `dwd_pbp_standard_element_type_coefficient_v` (
  `id` COMMENT "主键ID",
  `project_id` COMMENT "项目ID",
  `element_type_quantity_id` COMMENT "构件工程量ID",
  `quantity_name` COMMENT "工程量名称",
  `quantity_code` COMMENT "工程量编号",
  `coefficient` COMMENT "系数",
  `measure_unit` COMMENT "计量单位",
  `is_deleted` COMMENT "是否删除",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modified_id` COMMENT "系统-最后修改人"
) AS
SELECT
  `ods`.`a`.`id`,
  `ods`.`a`.`project_id`,
  `ods`.`a`.`sub_item_element_type_id` AS `element_type_quantity_id`,
  `ods`.`a`.`name` AS `quantity_name`,
  `ods`.`a`.`code` AS `quantity_code`,
  `ods`.`a`.`factor` AS `coefficient`,
  `ods`.`a`.`unit` AS `measure_unit`,
  `ods`.`a`.`deleted` AS `is_deleted`,
  `ods`.`a`.`sys_create_time`,
  `ods`.`a`.`sys_modified_time`,
  `ods`.`a`.`sys_creator_id`,
  `ods`.`a`.`sys_modified_id`
FROM
  `ods`.`ods_pbp_gpbp_pbs_sub_item_element_type_quantity` AS `a`
WHERE
  `ods`.`a`.`deleted` = 0;
--
CREATE VIEW `dwd_pbp_standard_element_type_quantity_v` (
  `id` COMMENT "主键ID",
  `project_id` COMMENT "项目ID",
  `sub_item_id` COMMENT "分部分项ID",
  `element_type_id` COMMENT "构件类型ID",
  `element_type_name` COMMENT "构件类型名称",
  `quantity_names` COMMENT "工程量名称集合",
  `quantity_expression` COMMENT "工程量表达式",
  `measure_unit` COMMENT "计量单位",
  `is_deleted` COMMENT "是否删除",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modified_id` COMMENT "系统-最后修改人",
  `system_type_code` COMMENT "系统类型编码",
  `system_type_name` COMMENT "系统类型名称"
) AS
SELECT
  `ods`.`a`.`id`,
  `ods`.`a`.`project_id`,
  `ods`.`a`.`sub_item_id`,
  `ods`.`a`.`element_type_id`,
  `ods`.`a`.`element_type_name`,
  `ods`.`a`.`quantity_name` AS `quantity_names`,
  `ods`.`a`.`quantities_expression` AS `quantity_expression`,
  `ods`.`a`.`unit` AS `measure_unit`,
  `ods`.`a`.`deleted` AS `is_deleted`,
  `ods`.`a`.`sys_create_time`,
  `ods`.`a`.`sys_modified_time`,
  `ods`.`a`.`sys_creator_id`,
  `ods`.`a`.`sys_modified_id`,
  `ods`.`a`.`system_type_code`,
  `ods`.`a`.`system_type_name`
FROM
  `ods`.`ods_pbp_gpbp_pbs_sub_item_element_type` AS `a`
WHERE
  `ods`.`a`.`deleted` = 0;
--
CREATE VIEW `dwd_pbp_standard_sub_item_v` (
  `id` COMMENT "主键ID",
  `parent_id` COMMENT "父ID",
  `full_id` COMMENT "全路径ID",
  `project_id` COMMENT "项目ID",
  `sub_item_code` COMMENT "分部分项编码",
  `sub_item_name` COMMENT "分部分项名称",
  `level_type` COMMENT "层级类型",
  `order_no` COMMENT "序号",
  `measure_unit` COMMENT "计量单位",
  `job_type_code` COMMENT "工种编码",
  `job_type_id` COMMENT "工种ID",
  `divide_desc` COMMENT "划分说明",
  `is_deleted` COMMENT "是否删除",
  `remark` COMMENT "备注",
  `source_id` COMMENT "来源ID",
  `data_source` COMMENT "数据来源",
  `construct_type_id` COMMENT "行业类型ID",
  `construct_type_code` COMMENT "行业类型编码",
  `specialty_type_id` COMMENT "专业类型ID",
  `specialty_type_code` COMMENT "专业类型编码",
  `job_type_name` COMMENT "工种名称",
  `job_type_catalog_id` COMMENT "工种分组ID",
  `measure_unit_code` COMMENT "计量单位编码",
  `job_type_catalog_code` COMMENT "工种分组编码",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modified_id` COMMENT "系统-最后修改人"
) AS
SELECT
  `ods`.`a`.`id`,
  `ods`.`a`.`pid` AS `parent_id`,
  `ods`.`a`.`full_id`,
  `ods`.`a`.`project_id`,
  `ods`.`a`.`code` AS `sub_item_code`,
  `ods`.`a`.`name` AS `sub_item_name`,
  `ods`.`a`.`level_type`,
  `ods`.`a`.`sub_order_no` AS `order_no`,
  `ods`.`a`.`unit` AS `measure_unit`,
  `ods`.`a`.`work_type_code` AS `job_type_code`,
  `ods`.`a`.`work_type_id` AS `job_type_id`,
  `ods`.`a`.`divide_explain` AS `divide_desc`,
  `ods`.`a`.`deleted` AS `is_deleted`,
  `ods`.`a`.`remark`,
  `ods`.`a`.`source_id`,
  `ods`.`a`.`source` AS `data_source`,
  `ods`.`a`.`construct_type_id`,
  `ods`.`a`.`construct_type_code`,
  `ods`.`a`.`speciality_type_id` AS `specialty_type_id`,
  `ods`.`a`.`speciality_type_code` AS `specialty_type_code`,
  `ods`.`a`.`work_type_name` AS `job_type_name`,
  `ods`.`a`.`work_type_catalog_id` AS `job_type_catalog_id`,
  `ods`.`a`.`unit_code` AS `measure_unit_code`,
  `ods`.`a`.`work_type_catalog_code` AS `job_type_catalog_code`,
  `ods`.`a`.`sys_create_time`,
  `ods`.`a`.`sys_modified_time`,
  `ods`.`a`.`sys_creator_id`,
  `ods`.`a`.`sys_modified_id`
FROM
  `ods`.`ods_pbp_gpbp_pbs_sub_item` AS `a`;
--
CREATE VIEW `dws_pbp_pbs_sub_item_element_type_detail_v` (
  `sub_item_id` COMMENT "分部ID/子分部ID",
  `project_id` COMMENT "项目ID",
  `element_type_id` COMMENT "构件类型ID",
  `quantity_expression` COMMENT "工程量表达式",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  DISTINCT `gpsiall`.`sub_item_id`,
  `gpsiall`.`project_id`,
  `dwd`.`gpsiet`.`element_type_id`,
  `dwd`.`psetc`.`quantity_code` AS `quantity_expression`,
  CASE
    WHEN ((length(`dwd`.`gpsiet`.`system_type_code`)) = 0) THEN -1
    WHEN (`dwd`.`gpsiet`.`system_type_code` IS NULL) THEN -1
    ELSE `dwd`.`gpsiet`.`system_type_code`
  END AS `system_type`
FROM
  (
    SELECT
      `dwd`.`gpsi`.`id` AS `sub_item_id`,
      `dwd`.`gpsi`.`project_id`,
      `dwd`.`gpsi2`.`id` AS `sub_entry_id`
    FROM
      `dwd`.`dwd_pbp_standard_sub_item_v` AS `gpsi`
      INNER JOIN `dwd`.`dwd_pbp_standard_sub_item_v` AS `gpsi2` ON (
        `dwd`.`gpsi`.`project_id` = `dwd`.`gpsi2`.`project_id`
      )
      AND (
        (
          starts_with(`dwd`.`gpsi2`.`full_id`, `dwd`.`gpsi`.`full_id`)
        ) = 1
      )
    WHERE
      (
        (
          (
            (`dwd`.`gpsi`.`level_type` = 101)
            OR (`dwd`.`gpsi`.`level_type` = 103)
          )
          AND (`dwd`.`gpsi2`.`level_type` = 102)
        )
        AND (`dwd`.`gpsi`.`is_deleted` = 0)
      )
      AND (`dwd`.`gpsi2`.`is_deleted` = 0)
  ) gpsiall
  LEFT OUTER JOIN `dwd`.`dwd_pbp_standard_element_type_quantity_v` AS `gpsiet` ON (
    (
      `gpsiall`.`project_id` = `dwd`.`gpsiet`.`project_id`
    )
    AND (
      `gpsiall`.`sub_entry_id` = `dwd`.`gpsiet`.`sub_item_id`
    )
  )
  AND (`dwd`.`gpsiet`.`is_deleted` = 0)
  LEFT OUTER JOIN `dwd`.`dwd_pbp_standard_element_type_coefficient_v` AS `psetc` ON (
    (
      (CAST(`gpsiall`.`project_id` AS BIGINT)) = `dwd`.`psetc`.`project_id`
    )
    AND (
      `dwd`.`gpsiet`.`id` = `dwd`.`psetc`.`element_type_quantity_id`
    )
  )
  AND (`dwd`.`psetc`.`is_deleted` = 0);
--
CREATE VIEW `dwd_pbp_pbs_segment_v` (
  `id` COMMENT "主键ID",
  `project_id` COMMENT "项目ID",
  `segment_full_id` COMMENT "施工段全路径ID",
  `segment_group_id` COMMENT "施工段分组ID",
  `building_id` COMMENT "单体ID",
  `specialty_id` COMMENT "专业ID",
  `segment_code` COMMENT "施工段编码",
  `segment_name` COMMENT "施工段名称",
  `start_floor_id` COMMENT "起始楼层ID",
  `end_floor_id` COMMENT "终止楼层ID",
  `start_elevation` COMMENT "起始标高",
  `end_elevation` COMMENT "终止标高",
  `position_type` COMMENT "位置类型",
  `dimension_type` COMMENT "维度类型",
  `color` COMMENT "颜色",
  `line_width` COMMENT "线宽",
  `remark` COMMENT "备注",
  `segment_line_type` COMMENT "线框类型",
  `task_status` COMMENT "任务状态",
  `task_deviation` COMMENT "任务偏差",
  `plan_start_time` COMMENT "计划开始时间",
  `plan_finish_time` COMMENT "计划完成时间",
  `actual_start_time` COMMENT "实际开始时间",
  `actual_finish_time` COMMENT "实际完成时间",
  `is_show_wireframe` COMMENT "是否显示线框",
  `is_need_label` COMMENT "需要调整标记",
  `is_link_edo` COMMENT "是否关联图元",
  `is_link_task` COMMENT "是否关联任务",
  `version` COMMENT "版本号",
  `order_no` COMMENT "序号",
  `sys_revision` COMMENT "系统-乐观锁版本号",
  `is_deleted` COMMENT "是否删除",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modifier_id` COMMENT "系统-最后修改人",
  `segment_area` COMMENT "施工段建筑面积",
  `segment_area_modification` COMMENT "施工段建筑面积修正值",
  `link_type` COMMENT "关联类型",
  `construction_phase_id` COMMENT "施工阶段ID",
  `model_segment_id` COMMENT "模型施工段ID",
  `item_id` COMMENT "分部ID",
  `sub_item_id` COMMENT "子分部ID",
  `table_type` COMMENT "数据所属表编码"
) AS
SELECT
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`full_id` AS `segment_full_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`segment_group_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`building_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`code` AS `segment_code`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`start_floor_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`end_floor_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`start_elevation`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`end_elevation`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`position_type`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`dimension_type`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`color`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`line_width`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`remark`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`segment_line_type`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`task_status`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`task_deviation`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`plan_start_time`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`plan_finish_time`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`real_start_time` AS `actual_start_time`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`real_finis_time` AS `actual_finish_time`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`is_show_line` AS `is_show_wireframe`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`is_need_relink` AS `is_need_label`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`is_related_edo` AS `is_link_edo`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`is_related_task` AS `is_link_task`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`version`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`order_no`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`area` AS `segment_area`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`area_modification` AS `segment_area_modification`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`relate_type` AS `link_type`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`model_construct_id` AS `model_segment_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`item_id`,
  `ods`.`ods_pbp_segment_0_bim_10_0_new`.`sub_item_id`,
  '0_bim_202005181910_0' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_0_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`full_id` AS `segment_full_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`segment_group_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`building_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`code` AS `segment_code`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`start_floor_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`end_floor_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`start_elevation`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`end_elevation`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`position_type`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`dimension_type`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`color`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`line_width`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`remark`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`segment_line_type`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`task_status`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`task_deviation`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`plan_start_time`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`plan_finish_time`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`real_start_time` AS `actual_start_time`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`real_finis_time` AS `actual_finish_time`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`is_show_line` AS `is_show_wireframe`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`is_need_relink` AS `is_need_label`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`is_related_edo` AS `is_link_edo`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`is_related_task` AS `is_link_task`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`version`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`order_no`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`area` AS `segment_area`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`area_modification` AS `segment_area_modification`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`relate_type` AS `link_type`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`model_construct_id` AS `model_segment_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`item_id`,
  `ods`.`ods_pbp_segment_1_bim_10_0_new`.`sub_item_id`,
  '1_bim_202005181910_0' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_1_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`full_id` AS `segment_full_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`segment_group_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`building_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`code` AS `segment_code`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`start_floor_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`end_floor_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`start_elevation`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`end_elevation`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`position_type`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`dimension_type`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`color`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`line_width`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`remark`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`segment_line_type`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`task_status`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`task_deviation`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`plan_start_time`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`plan_finish_time`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`real_start_time` AS `actual_start_time`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`real_finis_time` AS `actual_finish_time`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`is_show_line` AS `is_show_wireframe`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`is_need_relink` AS `is_need_label`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`is_related_edo` AS `is_link_edo`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`is_related_task` AS `is_link_task`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`version`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`order_no`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`area` AS `segment_area`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`area_modification` AS `segment_area_modification`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`relate_type` AS `link_type`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`model_construct_id` AS `model_segment_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`item_id`,
  `ods`.`ods_pbp_segment_2_bim_10_1_new`.`sub_item_id`,
  '2_bim_202005181910_1' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_2_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`full_id` AS `segment_full_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`segment_group_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`building_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`code` AS `segment_code`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`start_floor_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`end_floor_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`start_elevation`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`end_elevation`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`position_type`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`dimension_type`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`color`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`line_width`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`remark`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`segment_line_type`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`task_status`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`task_deviation`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`plan_start_time`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`plan_finish_time`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`real_start_time` AS `actual_start_time`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`real_finis_time` AS `actual_finish_time`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`is_show_line` AS `is_show_wireframe`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`is_need_relink` AS `is_need_label`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`is_related_edo` AS `is_link_edo`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`is_related_task` AS `is_link_task`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`version`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`order_no`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`area` AS `segment_area`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`area_modification` AS `segment_area_modification`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`relate_type` AS `link_type`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`model_construct_id` AS `model_segment_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`item_id`,
  `ods`.`ods_pbp_segment_3_bim_10_1_new`.`sub_item_id`,
  '3_bim_202005181910_1' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_3_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`full_id` AS `segment_full_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`segment_group_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`building_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`code` AS `segment_code`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`start_floor_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`end_floor_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`start_elevation`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`end_elevation`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`position_type`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`dimension_type`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`color`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`line_width`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`remark`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`segment_line_type`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`task_status`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`task_deviation`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`plan_start_time`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`plan_finish_time`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`real_start_time` AS `actual_start_time`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`real_finis_time` AS `actual_finish_time`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`is_show_line` AS `is_show_wireframe`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`is_need_relink` AS `is_need_label`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`is_related_edo` AS `is_link_edo`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`is_related_task` AS `is_link_task`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`version`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`order_no`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`area` AS `segment_area`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`area_modification` AS `segment_area_modification`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`relate_type` AS `link_type`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`model_construct_id` AS `model_segment_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`item_id`,
  `ods`.`ods_pbp_segment_0_bim_30_0_new`.`sub_item_id`,
  '0_bim_1120655100311625730_0' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_0_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`full_id` AS `segment_full_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`segment_group_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`building_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`code` AS `segment_code`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`start_floor_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`end_floor_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`start_elevation`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`end_elevation`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`position_type`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`dimension_type`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`color`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`line_width`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`remark`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`segment_line_type`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`task_status`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`task_deviation`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`plan_start_time`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`plan_finish_time`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`real_start_time` AS `actual_start_time`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`real_finis_time` AS `actual_finish_time`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`is_show_line` AS `is_show_wireframe`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`is_need_relink` AS `is_need_label`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`is_related_edo` AS `is_link_edo`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`is_related_task` AS `is_link_task`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`version`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`order_no`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`area` AS `segment_area`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`area_modification` AS `segment_area_modification`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`relate_type` AS `link_type`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`model_construct_id` AS `model_segment_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`item_id`,
  `ods`.`ods_pbp_segment_1_bim_30_0_new`.`sub_item_id`,
  '1_bim_1120655100311625730_0' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_1_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`full_id` AS `segment_full_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`segment_group_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`building_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`code` AS `segment_code`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`start_floor_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`end_floor_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`start_elevation`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`end_elevation`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`position_type`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`dimension_type`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`color`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`line_width`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`remark`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`segment_line_type`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`task_status`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`task_deviation`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`plan_start_time`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`plan_finish_time`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`real_start_time` AS `actual_start_time`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`real_finis_time` AS `actual_finish_time`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`is_show_line` AS `is_show_wireframe`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`is_need_relink` AS `is_need_label`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`is_related_edo` AS `is_link_edo`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`is_related_task` AS `is_link_task`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`version`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`order_no`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`area` AS `segment_area`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`area_modification` AS `segment_area_modification`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`relate_type` AS `link_type`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`model_construct_id` AS `model_segment_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`item_id`,
  `ods`.`ods_pbp_segment_2_bim_30_1_new`.`sub_item_id`,
  '2_bim_1120655100311625730_1' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_2_bim_30_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`full_id` AS `segment_full_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`segment_group_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`building_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`code` AS `segment_code`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`start_floor_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`end_floor_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`start_elevation`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`end_elevation`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`position_type`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`dimension_type`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`color`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`line_width`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`remark`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`segment_line_type`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`task_status`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`task_deviation`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`plan_start_time`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`plan_finish_time`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`real_start_time` AS `actual_start_time`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`real_finis_time` AS `actual_finish_time`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`is_show_line` AS `is_show_wireframe`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`is_need_relink` AS `is_need_label`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`is_related_edo` AS `is_link_edo`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`is_related_task` AS `is_link_task`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`version`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`order_no`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`area` AS `segment_area`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`area_modification` AS `segment_area_modification`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`relate_type` AS `link_type`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`model_construct_id` AS `model_segment_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`item_id`,
  `ods`.`ods_pbp_segment_3_bim_30_1_new`.`sub_item_id`,
  '3_bim_1120655100311625730_1' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_3_bim_30_1_new`;
--
CREATE VIEW `dim_pbp_pbs_segment_group_v` (
  `id` COMMENT "主键ID",
  `project_id` COMMENT "项目ID",
  `group_full_id` COMMENT "分组全路径ID",
  `parent_id` COMMENT "父ID",
  `specialty_id` COMMENT "专业ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `structure_code` COMMENT "结构编码",
  `structure_name` COMMENT "结构名称",
  `remark` COMMENT "备注",
  `structure_type` COMMENT "结构类型",
  `order_no` COMMENT "序号",
  `version` COMMENT "版本号",
  `is_deleted` COMMENT "是否删除",
  `sys_revision` COMMENT "系统-乐观锁版本号",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modifier_id` COMMENT "系统-最后修改人",
  `construction_phase_id` COMMENT "施工阶段ID",
  `third_part_id` COMMENT "第三方ID",
  `third_part_parent_id` COMMENT "第三方父ID",
  `item_id` COMMENT "分部ID",
  `sub_item_id` COMMENT "子分部ID",
  `table_type` COMMENT "数据所属表编码"
) AS
SELECT
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`full_id` AS `group_full_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`parent_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`building_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`floor_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`code` AS `structure_code`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`name` AS `structure_name`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`remark`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`type` AS `structure_type`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`order_no`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`version`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`f_unit_id` AS `third_part_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`f_unit_pid` AS `third_part_parent_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`item_id`,
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`.`sub_item_id`,
  '0_bim_202005181910_0' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_group_0_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`full_id` AS `group_full_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`parent_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`building_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`floor_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`code` AS `structure_code`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`name` AS `structure_name`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`remark`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`type` AS `structure_type`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`order_no`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`version`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`f_unit_id` AS `third_part_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`f_unit_pid` AS `third_part_parent_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`item_id`,
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`.`sub_item_id`,
  '1_bim_202005181910_0' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_group_1_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`full_id` AS `group_full_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`parent_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`building_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`floor_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`code` AS `structure_code`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`name` AS `structure_name`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`remark`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`type` AS `structure_type`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`order_no`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`version`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`f_unit_id` AS `third_part_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`f_unit_pid` AS `third_part_parent_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`item_id`,
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`.`sub_item_id`,
  '2_bim_202005181910_1' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_group_2_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`full_id` AS `group_full_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`parent_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`building_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`floor_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`code` AS `structure_code`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`name` AS `structure_name`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`remark`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`type` AS `structure_type`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`order_no`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`version`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`f_unit_id` AS `third_part_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`f_unit_pid` AS `third_part_parent_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`item_id`,
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`.`sub_item_id`,
  '3_bim_202005181910_1' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_group_3_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`full_id` AS `group_full_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`parent_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`building_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`floor_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`code` AS `structure_code`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`name` AS `structure_name`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`remark`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`type` AS `structure_type`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`order_no`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`version`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`f_unit_id` AS `third_part_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`f_unit_pid` AS `third_part_parent_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`item_id`,
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`.`sub_item_id`,
  '0_bim_1120655100311625730_0' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_group_0_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`full_id` AS `group_full_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`parent_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`building_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`floor_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`code` AS `structure_code`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`name` AS `structure_name`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`remark`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`type` AS `structure_type`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`order_no`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`version`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`f_unit_id` AS `third_part_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`f_unit_pid` AS `third_part_parent_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`item_id`,
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`.`sub_item_id`,
  '1_bim_1120655100311625730_0' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_group_1_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`full_id` AS `group_full_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`parent_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`building_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`floor_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`code` AS `structure_code`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`name` AS `structure_name`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`remark`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`type` AS `structure_type`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`order_no`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`version`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`f_unit_id` AS `third_part_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`f_unit_pid` AS `third_part_parent_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`item_id`,
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`.`sub_item_id`,
  '2_bim_1120655100311625730_1' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_group_2_bim_30_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`full_id` AS `group_full_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`parent_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`specialty_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`building_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`floor_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`code` AS `structure_code`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`name` AS `structure_name`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`remark`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`type` AS `structure_type`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`order_no`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`version`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`is_delete` AS `is_deleted`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`sys_revision`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`sys_modified_time`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`sys_modifier_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`construction_phase_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`f_unit_id` AS `third_part_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`f_unit_pid` AS `third_part_parent_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`item_id`,
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`.`sub_item_id`,
  '3_bim_1120655100311625730_1' AS `table_type`
FROM
  `ods`.`ods_pbp_segment_group_3_bim_30_1_new`;
--
CREATE VIEW `dim_pbp_pbs_model_segment_v` (
  `id` COMMENT "主键ID",
  `segment_name` COMMENT "施工段名称",
  `project_id` COMMENT "项目ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `phase_id` COMMENT "施工阶段ID",
  `phase_name` COMMENT "施工阶段名称",
  `is_deleted` COMMENT "是否删除",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `construct_no` COMMENT "施工段编号",
  `is_rollup` COMMENT "是否上卷",
  `table_type` COMMENT "数据所属表编码"
) AS
SELECT
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`building_id`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`floor_id`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`phase_id`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`phase_name`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`deleted` AS `is_deleted`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`construct_id` AS `construct_no`,
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`.`show_label` AS `is_rollup`,
  '0_bim_202005181910_0' AS `table_type`
FROM
  `ods`.`ods_pbp_model_construction_0_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`building_id`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`floor_id`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`phase_id`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`phase_name`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`deleted` AS `is_deleted`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`construct_id` AS `construct_no`,
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`.`show_label` AS `is_rollup`,
  '0_bim_1120655100311625730_0' AS `table_type`
FROM
  `ods`.`ods_pbp_model_construction_0_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`id`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`project_id`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`building_id`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`floor_id`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`phase_id`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`phase_name`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`deleted` AS `is_deleted`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`construct_id` AS `construct_no`,
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`.`show_label` AS `is_rollup`,
  '1_bim_202005181910_0' AS `table_type`
FROM
  `ods`.`ods_pbp_model_construction_1_bim_10_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`id`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`project_id`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`building_id`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`floor_id`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`phase_id`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`phase_name`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`deleted` AS `is_deleted`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`sys_create_time`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`sys_creator_id`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`construct_id` AS `construct_no`,
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`.`show_label` AS `is_rollup`,
  '1_bim_1120655100311625730_0' AS `table_type`
FROM
  `ods`.`ods_pbp_model_construction_1_bim_30_0_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`building_id`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`floor_id`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`phase_id`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`phase_name`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`deleted` AS `is_deleted`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`construct_id` AS `construct_no`,
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`.`show_label` AS `is_rollup`,
  '2_bim_202005181910_1' AS `table_type`
FROM
  `ods`.`ods_pbp_model_construction_2_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`building_id`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`floor_id`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`phase_id`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`phase_name`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`deleted` AS `is_deleted`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`construct_id` AS `construct_no`,
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`.`show_label` AS `is_rollup`,
  '2_bim_1120655100311625730_1' AS `table_type`
FROM
  `ods`.`ods_pbp_model_construction_2_bim_30_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`id`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`project_id`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`building_id`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`floor_id`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`phase_id`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`phase_name`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`deleted` AS `is_deleted`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`construct_id` AS `construct_no`,
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`.`show_label` AS `is_rollup`,
  '3_bim_202005181910_1' AS `table_type`
FROM
  `ods`.`ods_pbp_model_construction_3_bim_10_1_new`
UNION ALL
SELECT
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`id`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`name` AS `segment_name`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`project_id`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`building_id`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`floor_id`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`phase_id`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`phase_name`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`deleted` AS `is_deleted`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`sys_create_time`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`sys_creator_id`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`construct_id` AS `construct_no`,
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`.`show_label` AS `is_rollup`,
  '3_bim_1120655100311625730_1' AS `table_type`
FROM
  `ods`.`ods_pbp_model_construction_3_bim_30_1_new`;
--
CREATE VIEW `dwd_pbp_pbs_structure_v` (
  `id` COMMENT "主键ID",
  `project_id` COMMENT "项目ID",
  `full_id` COMMENT "全路径ID",
  `parent_id` COMMENT "父ID",
  `specialty_id` COMMENT "专业ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `structure_code` COMMENT "结构编码",
  `structure_name` COMMENT "结构名称",
  `remark` COMMENT "备注",
  `order_no` COMMENT "序号",
  `version` COMMENT "版本号",
  `is_deleted` COMMENT "是否删除",
  `sys_revision` COMMENT "系统-乐观锁版本号",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modifier_id` COMMENT "系统-最后修改人",
  `construction_phase_id` COMMENT "施工阶段ID",
  `structure_type` COMMENT "结构类型",
  `model_segment_id` COMMENT "模型施工段ID",
  `table_type` COMMENT "数据所属表编码"
) AS WITH cte(
  `id`,
  `project_id`,
  `full_id`,
  `parent_id`,
  `specialty_id`,
  `building_id`,
  `floor_id`,
  `structure_code`,
  `structure_name`,
  `remark`,
  `order_no`,
  `version`,
  `is_deleted`,
  `sys_revision`,
  `sys_create_time`,
  `sys_modified_time`,
  `sys_creator_id`,
  `sys_modifier_id`,
  `construction_phase_id`,
  `structure_type`,
  `model_segment_id`,
  `table_type`
) AS (
  SELECT
    `dim`.`g`.`id`,
    `dim`.`d`.`project_id`,
    `dim`.`g`.`group_full_id` AS `full_id`,
    NULL AS `parent_id`,
    NULL AS `specialty_id`,
    `dim`.`d`.`building_id`,
    `dim`.`d`.`floor_id`,
    NULL AS `structure_code`,
    `dim`.`d`.`segment_name` AS `structure_name`,
    NULL AS `remark`,
    NULL AS `order_no`,
    NULL AS `version`,
    0 AS `is_deleted`,
    NULL AS `sys_revision`,
    NULL AS `sys_create_time`,
    NULL AS `sys_modified_time`,
    NULL AS `sys_creator_id`,
    NULL AS `sys_modifier_id`,
    `dim`.`d`.`phase_id` AS `construction_phase_id`,
    5 AS `structure_type`,
    `dim`.`d`.`id` AS `model_segment_id`,
    `dim`.`d`.`table_type`
  FROM
    `dim`.`dim_pbp_pbs_model_segment_v` AS `d`
    INNER JOIN `dim`.`dim_pbp_pbs_segment_group_v` AS `g` ON (
      (
        (
          (
            (`dim`.`g`.`is_deleted` = 0)
            AND (`dim`.`d`.`project_id` = `dim`.`g`.`project_id`)
          )
          AND (
            `dim`.`d`.`building_id` = `dim`.`g`.`building_id`
          )
        )
        AND (`dim`.`d`.`floor_id` = `dim`.`g`.`floor_id`)
      )
      AND (
        `dim`.`d`.`phase_id` = `dim`.`g`.`construction_phase_id`
      )
    )
    AND (`dim`.`d`.`table_type` = `dim`.`g`.`table_type`)
    LEFT OUTER JOIN `dwd`.`dwd_pbp_pbs_segment_v` AS `s` ON (
      (`dim`.`d`.`id` = `dwd`.`s`.`model_segment_id`)
      AND (`dim`.`d`.`project_id` = `dwd`.`s`.`project_id`)
    )
    AND (`dim`.`d`.`table_type` = `dwd`.`s`.`table_type`)
  WHERE
    (
      (
        (`dim`.`d`.`is_rollup` = 0)
        AND (`dwd`.`s`.`id` IS NULL)
      )
      AND (`dim`.`d`.`segment_name` LIKE '未归类%')
    )
    AND (`dim`.`d`.`is_deleted` = 0)
  UNION ALL
  SELECT
    `dwd`.`b`.`id`,
    `dwd`.`b`.`project_id`,
    `dwd`.`b`.`segment_full_id` AS `full_id`,
    `dwd`.`b`.`segment_group_id` AS `parent_id`,
    `dwd`.`b`.`specialty_id`,
    `dwd`.`b`.`building_id`,
    `dwd`.`b`.`start_floor_id` AS `floor_id`,
    `dwd`.`b`.`segment_code` AS `structure_code`,
    `dwd`.`b`.`segment_name` AS `structure_name`,
    `dwd`.`b`.`remark`,
    `dwd`.`b`.`order_no`,
    `dwd`.`b`.`version`,
    `dwd`.`b`.`is_deleted`,
    `dwd`.`b`.`sys_revision`,
    `dwd`.`b`.`sys_create_time`,
    `dwd`.`b`.`sys_modified_time`,
    `dwd`.`b`.`sys_creator_id`,
    `dwd`.`b`.`sys_modifier_id`,
    `dwd`.`b`.`construction_phase_id`,
    5 AS `structure_type`,
    `dwd`.`b`.`model_segment_id`,
    `dwd`.`b`.`table_type`
  FROM
    `dwd`.`dwd_pbp_pbs_segment_v` AS `b`
),
pbs(
  `id`,
  `project_id`,
  `full_id`,
  `parent_id`,
  `specialty_id`,
  `building_id`,
  `floor_id`,
  `structure_code`,
  `structure_name`,
  `remark`,
  `order_no`,
  `version`,
  `is_deleted`,
  `sys_revision`,
  `sys_create_time`,
  `sys_modified_time`,
  `sys_creator_id`,
  `sys_modifier_id`,
  `construction_phase_id`,
  `structure_type`,
  `model_segment_id`,
  `table_type`
) AS (
  SELECT
    `dim`.`a`.`id`,
    `dim`.`a`.`project_id`,
    `dim`.`a`.`group_full_id` AS `full_id`,
    `dim`.`a`.`parent_id`,
    `dim`.`a`.`specialty_id`,
    `dim`.`a`.`building_id`,
    `dim`.`a`.`floor_id`,
    `dim`.`a`.`structure_code`,
    `dim`.`a`.`structure_name`,
    `dim`.`a`.`remark`,
    `dim`.`a`.`order_no`,
    `dim`.`a`.`version`,
    `dim`.`a`.`is_deleted`,
    `dim`.`a`.`sys_revision`,
    `dim`.`a`.`sys_create_time`,
    `dim`.`a`.`sys_modified_time`,
    `dim`.`a`.`sys_creator_id`,
    `dim`.`a`.`sys_modifier_id`,
    `dim`.`a`.`construction_phase_id`,
    `dim`.`a`.`structure_type`,
    NULL AS `model_segment_id`,
    `dim`.`a`.`table_type`
  FROM
    `dim`.`dim_pbp_pbs_segment_group_v` AS `a`
  UNION ALL
  SELECT
    `b`.`id`,
    `b`.`project_id`,
    `b`.`full_id`,
    `b`.`parent_id`,
    `b`.`specialty_id`,
    `b`.`building_id`,
    `b`.`floor_id`,
    `b`.`structure_code`,
    `b`.`structure_name`,
    `b`.`remark`,
    `b`.`order_no`,
    `b`.`version`,
    `b`.`is_deleted`,
    `b`.`sys_revision`,
    `b`.`sys_create_time`,
    `b`.`sys_modified_time`,
    `b`.`sys_creator_id`,
    `b`.`sys_modifier_id`,
    `b`.`construction_phase_id`,
    `b`.`structure_type`,
    `b`.`model_segment_id`,
    `b`.`table_type`
  FROM
    cte AS b
)
SELECT
  `pbs`.`id`,
  `pbs`.`project_id`,
  `pbs`.`full_id`,
  `pbs`.`parent_id`,
  `pbs`.`specialty_id`,
  `pbs`.`building_id`,
  `pbs`.`floor_id`,
  `pbs`.`structure_code`,
  `pbs`.`structure_name`,
  `pbs`.`remark`,
  `pbs`.`order_no`,
  `pbs`.`version`,
  `pbs`.`is_deleted`,
  `pbs`.`sys_revision`,
  `pbs`.`sys_create_time`,
  `pbs`.`sys_modified_time`,
  `pbs`.`sys_creator_id`,
  `pbs`.`sys_modifier_id`,
  `pbs`.`construction_phase_id`,
  `pbs`.`structure_type`,
  `pbs`.`model_segment_id`,
  `pbs`.`table_type`
FROM
  pbs;
--
CREATE VIEW `dwd_pbp_pbs_structure_fill_v` (
  `id` COMMENT "主键ID",
  `project_id` COMMENT "项目ID",
  `full_id` COMMENT "全路径ID",
  `parent_id` COMMENT "父ID",
  `specialty_id` COMMENT "专业ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `structure_code` COMMENT "结构编码",
  `structure_name` COMMENT "结构名称",
  `remark` COMMENT "备注",
  `order_no` COMMENT "序号",
  `version` COMMENT "版本号",
  `is_deleted` COMMENT "是否删除",
  `sys_revision` COMMENT "系统-乐观锁版本号",
  `sys_create_time` COMMENT "系统-记录创建时间",
  `sys_modified_time` COMMENT "系统-记录修改时间",
  `sys_creator_id` COMMENT "系统-创建人",
  `sys_modifier_id` COMMENT "系统-最后修改人",
  `construction_phase_id` COMMENT "施工阶段ID",
  `structure_type` COMMENT "结构类型",
  `model_segment_id` COMMENT "模型施工段ID",
  `table_type` COMMENT "数据所属表编码"
) AS
SELECT
  `dwd`.`s`.`id`,
  `dwd`.`s`.`project_id`,
  `dwd`.`s`.`full_id`,
  `dwd`.`s`.`parent_id`,
  `dwd`.`s`.`specialty_id`,
  ifnull(`dwd`.`s`.`building_id`, 0) AS `building_id`,
  ifnull(`dwd`.`s`.`floor_id`, 0) AS `floor_id`,
  `dwd`.`s`.`structure_code`,
  `dwd`.`s`.`structure_name`,
  `dwd`.`s`.`remark`,
  `dwd`.`s`.`order_no`,
  `dwd`.`s`.`version`,
  `dwd`.`s`.`is_deleted`,
  `dwd`.`s`.`sys_revision`,
  `dwd`.`s`.`sys_create_time`,
  `dwd`.`s`.`sys_modified_time`,
  `dwd`.`s`.`sys_creator_id`,
  `dwd`.`s`.`sys_modifier_id`,
  ifnull(`dwd`.`s`.`construction_phase_id`, 0) AS `construction_phase_id`,
  `dwd`.`s`.`structure_type`,
  ifnull(`dwd`.`s`.`model_segment_id`, 0) AS `model_segment_id`,
  `dwd`.`s`.`table_type`
FROM
  `dwd`.`dwd_pbp_pbs_structure_v` AS `s`;
--
CREATE VIEW `dwd_pbp_calculate_element_model_quantity_v` (
  `project_id` COMMENT "项目ID",
  `sub_special_id` COMMENT "专业ID",
  `region_id` COMMENT "区域ID",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `construction_phase_id` COMMENT "施工阶段ID",
  `segment_id` COMMENT "施工段ID",
  `original_floor_id` COMMENT "原始楼层ID",
  `original_construction_phase_id` COMMENT "原始施工阶段ID",
  `original_segment_id` COMMENT "原始施工段ID",
  `element_id` COMMENT "构件ID",
  `element_name` COMMENT "构件名称",
  `element_type_id` COMMENT "构件类型",
  `element_type_name` COMMENT "构件类型名称",
  `quantity_type_code` COMMENT "算量类型编码",
  `quantity_expression` COMMENT "算量表达式",
  `quantity_name` COMMENT "算量名称",
  `quantity_value` COMMENT "算量值",
  `measure_unit` COMMENT "算量单位",
  `model_specification` COMMENT "模型规格",
  `system_type` COMMENT "系统类型"
) AS
SELECT
  `gtcp_quantity`.`q`.`project_id`,
  `gtcp_quantity`.`q`.`sub_special_id`,
  `gtcp_quantity`.`q`.`region_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_building_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_building_id`
  END AS `building_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_floor_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_floor_id`
  END AS `floor_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_construct_phase_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_construct_phase_id`
  END AS `construction_phase_id`,
  CASE
    WHEN (`gtcp_quantity`.`q`.`pf_construct_id` = -1) THEN 0
    ELSE `gtcp_quantity`.`q`.`pf_construct_id`
  END AS `segment_id`,
  `gtcp_quantity`.`q`.`floor_id` AS `original_floor_id`,
  `gtcp_quantity`.`q`.`construct_type_id` AS `original_construction_phase_id`,
  `gtcp_quantity`.`q`.`construct_id` AS `original_segment_id`,
  `gtcp_quantity`.`q`.`element_id`,
  `gtcp_quantity`.`q`.`element_name`,
  `gtcp_quantity`.`q`.`element_type_id`,
  `gtcp_quantity`.`q`.`element_type_name`,
  `gtcp_quantity`.`q`.`phase_id` AS `quantity_type_code`,
  `gtcp_quantity`.`q`.`qty_code` AS `quantity_expression`,
  `gtcp_quantity`.`q`.`qty_name` AS `quantity_name`,
  `gtcp_quantity`.`q`.`qty_value` AS `quantity_value`,
  `gtcp_quantity`.`q`.`unit` AS `measure_unit`,
  `gtcp_quantity`.`q`.`model_spec` AS `model_specification`,
  -1 AS `system_type`
FROM
  `gtcp_quantity`.`gtc_ft_element_model_quantity` AS `q`;
--
CREATE VIEW `dws_pbp_pbs_model_quantity_detail_v` (
  `pbs_id` COMMENT "PBSID",
  `full_id` COMMENT "全路径ID",
  `table_type` COMMENT "数据所属表编码",
  `project_id` COMMENT "项目ID",
  `sub_special_id` COMMENT "专业ID",
  `quantity_type_code` COMMENT "算量类别编码",
  `building_id` COMMENT "单体ID",
  `floor_id` COMMENT "楼层ID",
  `construction_phase_id` COMMENT "施工阶段ID",
  `segment_id` COMMENT "施工段ID",
  `element_id` COMMENT "构件ID",
  `element_name` COMMENT "构件名称",
  `element_type_id` COMMENT "构件类型ID",
  `element_type_name` COMMENT "构件类型名称",
  `quantity_expression` COMMENT "算量表达式",
  `quantity_name` COMMENT "算量名称",
  `quantity_value` COMMENT "算量数值",
  `measure_unit` COMMENT "算量单位",
  `model_specification` COMMENT "模型规格"
) AS
SELECT
  `dwd`.`p`.`id` AS `pbs_id`,
  `dwd`.`p`.`full_id`,
  `dwd`.`p`.`table_type`,
  `dwd`.`q`.`project_id`,
  `dwd`.`q`.`sub_special_id`,
  `dwd`.`q`.`quantity_type_code`,
  `dwd`.`q`.`building_id`,
  `dwd`.`q`.`floor_id`,
  `dwd`.`q`.`construction_phase_id`,
  `dwd`.`q`.`segment_id`,
  `dwd`.`q`.`element_id`,
  `dwd`.`q`.`element_name`,
  `dwd`.`q`.`element_type_id`,
  `dwd`.`q`.`element_type_name`,
  `dwd`.`q`.`quantity_expression`,
  `dwd`.`q`.`quantity_name`,
  `dwd`.`q`.`quantity_value`,
  `dwd`.`q`.`measure_unit`,
  `dwd`.`q`.`model_specification`
FROM
  `dwd`.`dwd_pbp_calculate_element_model_quantity_v` AS `q`
  INNER JOIN `dwd`.`dwd_pbp_pbs_structure_fill_v` AS `p` ON (
    (
      (
        (`dwd`.`p`.`project_id` = `dwd`.`q`.`project_id`)
        AND (
          `dwd`.`p`.`building_id` = `dwd`.`q`.`building_id`
        )
      )
      AND (`dwd`.`p`.`floor_id` = `dwd`.`q`.`floor_id`)
    )
    AND (
      `dwd`.`p`.`construction_phase_id` = `dwd`.`q`.`construction_phase_id`
    )
  )
  AND (
    `dwd`.`p`.`model_segment_id` = `dwd`.`q`.`segment_id`
  )
WHERE
  `dwd`.`p`.`is_deleted` = 0;
~~~
