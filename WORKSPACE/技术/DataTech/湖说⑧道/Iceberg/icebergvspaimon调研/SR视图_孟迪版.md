整体关系
![[pms接口sr视图关系.png]]

# 项目PBS工程量统计接口
/gdmp/projects/%s/pbs_model_quantity_statistic
``` sql
select
    q.element_type_id as elementTypeId,
    q.element_type_name as elementTypeName,
    q.quantity_name as quantityName,
    q.quantity_expression as quantityExpression,
    q.measure_unit as measureUnit,
    round ( sum ( case when q.quantity_type_code = 1 then q.quantity_value else 0 end ), 6 ) as budget,
    round ( sum ( case when q.quantity_type_code = 2 then q.quantity_value else 0 end ), 6 ) as deepen ,
    round ( sum ( case when q.quantity_type_code = 4 then q.quantity_value else 0 end ), 6 ) as retrofit,
    round ( sum ( case when q.quantity_type_code = 5 then q.quantity_value else 0 end ), 6 ) as budgetInput,
    round ( sum ( case when q.quantity_type_code = 6 then q.quantity_value else 0 end ), 6 ) as deepenInput,
    round ( sum ( case when q.quantity_type_code = 7 then q.quantity_value else 0 end ), 6 ) as retrofitInput
from dws.dws_pbp_pbs_model_quantity_detail_v q
where q.project_id = # { projectid }
    # [ and q.full_id like ## { fullId } ] #
    # [ and q.pbs_id in ($$ { nodeIds }) ] #
    # [ and q.element_type_id = ## { elementTypeId } ] #
group by
 q.element_type_id,
 q.element_type_name,
 q.quantity_name,
 q.measure_unit,
 q.quantity_expression
order by
    q.element_type_name,
    q.quantity_name
```

## dws_pbp_pbs_model_quantity_detail_v
生产中台PBS工程量明细视图
``` sql
select
 p.id as pbs_id,
 p.full_id,
 p.table_type,
 q.project_id,
 q.sub_special_id,
 q.quantity_type_code,
 q.building_id,
 q.floor_id,
 q.construction_phase_id,
 q.segment_id,
 q.element_id,
 q.element_name,
 q.element_type_id,
 q.element_type_name,
 q.quantity_expression,
 q.quantity_name,
 q.quantity_value,
 q.measure_unit,
 q.model_specification
from dwd.dwd_pbp_calculate_element_model_quantity_v q
join dwd.dwd_pbp_pbs_structure_fill_v p
 on p.project_id = q.project_id
 and p.building_id = q.building_id
 and p.floor_id = q.floor_id
 and p.construction_phase_id = q.construction_phase_id
 and p.model_segment_id = q.segment_id
where p.is_deleted = 0 ;
```

### dwd_pbp_calculate_element_model_quantity_v
生产中台算量域构件工程量视图

``` sql
select 
 q.project_id as project_id ,
 q.sub_special_id as sub_special_id ,
 q.region_id as region_id ,
case when q.pf_building_id=-1 then 0 else q.pf_building_id end as building_id,
case when q.pf_floor_id=-1 then 0 else q.pf_floor_id end as floor_id,
case when q.pf_construct_phase_id=-1 then 0 else q.pf_construct_phase_id end as construction_phase_id,
case when q.pf_construct_id=-1 then 0 else q.pf_construct_id end as segment_id ,
 q.floor_id as original_floor_id ,
 q.construct_type_id as original_construction_phase_id ,
 q.construct_id as original_segment_id ,
 q.element_id as element_id ,
 q.element_name as element_name ,
 q.element_type_id as element_type_id ,
 q.element_type_name as element_type_name ,
 q.phase_id as quantity_type_code ,
 q.qty_code as quantity_expression ,
 q.qty_name as quantity_name ,
 q.qty_value as quantity_value ,
 q.unit as measure_unit ,
 q.model_spec as model_specification,
 -1 as system_type
from gtcp_quantity.gtc_ft_element_model_quantity q ;
```
#### gtcp_quantity.gtc_ft_element_model_quantity
### dwd_pbp_pbs_structure_fill_v
生产中台PBS域分解结构填充视图


``` sql
select 
 s.id,
 s.project_id,
 s.full_id,
 s.parent_id,
 s.specialty_id,
 ifnull(s.building_id,0) as building_id,
 ifnull(s.floor_id,0) as floor_id,
 s.structure_code,
 s.structure_name,
 s.remark,
 s.order_no,
 s.version,
 s.is_deleted,
 s.sys_revision,
 s.sys_create_time,
 s.sys_modified_time,
 s.sys_creator_id,
 s.sys_modifier_id,
 ifnull(s.construction_phase_id,0) as construction_phase_id,
 s.structure_type,
 ifnull(s.model_segment_id,0) as model_segment_id,
 s.table_type
from dwd.dwd_pbp_pbs_structure_v s
where s.is_deleted = 0
```

 构件工程量明细表
gtc_ft_element_model_quantity
CREATE TABLE `gtc_ft_element_model_quantity` (
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
  `element_id` largeint(40) NOT NULL COMMENT "构件ID",
  `qty_code` varchar(255) NOT NULL COMMENT "工程量Code",
  `qty_name` varchar(255) REPLACE NULL DEFAULT " " COMMENT "工程量名称",
  `element_type_name` varchar(255) REPLACE NULL DEFAULT " " COMMENT "构件类型名称",
  `element_name` varchar(512) REPLACE NULL DEFAULT "" COMMENT "构件名称",
  `qty_value` decimal(38, 6) REPLACE NULL DEFAULT "0.0" COMMENT "工程量",
  `unit` varchar(50) REPLACE NULL DEFAULT "" COMMENT "工程量单位",
  `model_spec` varchar(255) REPLACE NULL DEFAULT "" COMMENT "模型规格"
) ENGINE=OLAP 
AGGREGATE KEY(`project_id`, `phase_id`, `sub_special_id`, `pf_building_id`, `pf_floor_id`, `pf_construct_phase_id`, `pf_construct_id`, `region_id`, `floor_id`, `construct_type_id`, `construct_id`, `element_type_id`, `element_id`, `qty_code`)
COMMENT "构件工程量明细表"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 50 
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);

#### dwd_pbp_pbs_structure_v
生产中台PBS域分解结构视图

``` sql
with cte as (
    select
        g.id as id,
        d.project_id,
        g.group_full_id as full_id,
        null as parent_id,
        null as specialty_id,
        d.building_id as building_id,
        d.floor_id as floor_id,
        null as structure_code,
        d.segment_name as structure_name,
        null as remark,
        null as order_no,
        null as version,
        0 as is_deleted,
        null as sys_revision,
        null as sys_create_time,
        null as sys_modified_time,
        null as sys_creator_id,
        null as sys_modifier_id,
        d.phase_id as construction_phase_id,
        5 as structure_type,
        d.id as model_segment_id,
        d.table_type as table_type
    from dim.dim_pbp_pbs_model_segment_v d
    join dim.dim_pbp_pbs_segment_group_v g
            on g.is_deleted = 0
            and d.project_id = g.project_id
            and d.table_type = g.table_type
            and d.building_id = g.building_id
            and d.floor_id = g.floor_id
            and d.phase_id = g.construction_phase_id
    left join dwd.dwd_pbp_pbs_segment_v s
         on d.id=s.model_segment_id
         and d.project_id = s.project_id
         and d.table_type = s.table_type
    where d.is_rollup = 0
        and s.id is null
        and d.segment_name like '未归类%'
        and d.is_deleted = 0
    union all
    select
       b.id as id,
       b.project_id as project_id,
       b.segment_full_id as full_id,
       b.segment_group_id as parent_id,
       b.specialty_id as specialty_id,
       b.building_id as building_id,
       b.start_floor_id as floor_id,
       b.segment_code as structure_code,
       b.segment_name as structure_name,
       b.remark as remark,
       b.order_no as order_no,
       b.version as version,
       b.is_deleted as is_deleted,
       b.sys_revision as sys_revision,
       b.sys_create_time as sys_create_time,
       b.sys_modified_time as sys_modified_time,
       b.sys_creator_id as sys_creator_id,
       b.sys_modifier_id as sys_modifier_id,
       b.construction_phase_id as construction_phase_id,
       5 as structure_type,
       b.model_segment_id as model_segment_id,
       b.table_type as table_type
    from dwd.dwd_pbp_pbs_segment_v b
), pbs as (
 select
   a.id as id,
   a.project_id as project_id,
   a.group_full_id as full_id,
   a.parent_id as parent_id,
   a.specialty_id as specialty_id,
   a.building_id as building_id,
   a.floor_id as floor_id,
   a.structure_code as structure_code,
   a.structure_name as structure_name,
   a.remark as remark,
   a.order_no as order_no,
   a.version as version,
   a.is_deleted as is_deleted,
   a.sys_revision as sys_revision,
   a.sys_create_time as sys_create_time,
   a.sys_modified_time as sys_modified_time,
   a.sys_creator_id as sys_creator_id,
   a.sys_modifier_id as sys_modifier_id,
   a.construction_phase_id as construction_phase_id,
   a.structure_type as structure_type,
   null as model_segment_id,
   a.table_type as table_type
 from
   dim.dim_pbp_pbs_segment_group_v a
 union all
 select
   b.id as id,
   b.project_id as project_id,
   b.full_id as full_id,
   b.parent_id as parent_id,
   b.specialty_id as specialty_id,
   b.building_id as building_id,
   b.floor_id as floor_id,
   b.structure_code as structure_code,
   b.structure_name as structure_name,
   b.remark as remark,
   b.order_no as order_no,
   b.version as version,
   b.is_deleted as is_deleted,
   b.sys_revision as sys_revision,
   b.sys_create_time as sys_create_time,
   b.sys_modified_time as sys_modified_time,
   b.sys_creator_id as sys_creator_id,
   b.sys_modifier_id as sys_modifier_id,
   b.construction_phase_id as construction_phase_id,
   b.structure_type as structure_type,
   b.model_segment_id as model_segment_id,
   b.table_type as table_type
 from
   cte b
)
select
 pbs.id,
 pbs.project_id,
 pbs.full_id,
 pbs.parent_id,
 pbs.specialty_id,
 pbs.building_id,
 pbs.floor_id,
 pbs.structure_code,
 pbs.structure_name,
 pbs.remark,
 pbs.order_no,
 pbs.version,
 pbs.is_deleted,
 pbs.sys_revision,
 pbs.sys_create_time,
 pbs.sys_modified_time,
 pbs.sys_creator_id,
 pbs.sys_modifier_id,
 pbs.construction_phase_id,
 pbs.structure_type,
 pbs.model_segment_id,
 pbs.table_type as table_type
from pbs ;
```
##### dim_pbp_pbs_model_segment_v
生产中台模型施工段维表视图

```sql
select
 id,
 name as segment_name,
 project_id,
 building_id,
 floor_id,
 phase_id,
 phase_name,
 deleted as is_deleted,
 sys_create_time,
 sys_creator_id,
 construct_id as construct_no,
 show_label as is_rollup,
'0_bim_202005181910_0' as table_type
from ods.ods_pbp_model_construction_0_bim_10_0_new
union all 
select
 id,
 name as segment_name,
 project_id,
 building_id,
 floor_id,
 phase_id,
 phase_name,
 deleted as is_deleted,
 sys_create_time,
 sys_creator_id,
 construct_id as construct_no,
 show_label as is_rollup,
'1_bim_202005181910_1' as table_type
from ods.ods_pbp_model_construction_1_bim_10_1_new
union all 
select
 id,
 name as segment_name,
 project_id,
 building_id,
 floor_id,
 phase_id,
 phase_name,
 deleted as is_deleted,
 sys_create_time,
 sys_creator_id,
 construct_id as construct_no,
 show_label as is_rollup,
'0_bim_1120655100311625730_0' as table_type
from ods.ods_pbp_model_construction_0_bim_30_0_new
union all 
select
 id,
 name as segment_name,
 project_id,
 building_id,
 floor_id,
 phase_id,
 phase_name,
 deleted as is_deleted,
 sys_create_time,
 sys_creator_id,
 construct_id as construct_no,
 show_label as is_rollup,
'1_bim_1120655100311625730_1' as table_type
from ods.ods_pbp_model_construction_1_bim_30_1_new ;
```

###### ods.ods_pbp_model_construction_0_bim_10_0_new
##### dim_pbp_pbs_segment_group_v
生产中台施工段分组维表视图

```sql
select
  a.id as id,
  a.project_id as project_id,
  a.full_id as group_full_id,
  a.parent_id as parent_id,
  a.specialty_id as specialty_id,
  a.building_id as building_id,
  a.floor_id as floor_id,
  a.code as structure_code,
  a.name as structure_name,
  a.remark as remark,
  a.type as structure_type,
  a.order_no as order_no,
  a.version as version,
  a.is_delete as is_deleted,
  a.sys_revision as sys_revision,
  a.sys_create_time as sys_create_time,
  a.sys_modified_time as sys_modified_time,
  a.sys_creator_id as sys_creator_id,
  a.sys_modifier_id as sys_modifier_id,
  a.construction_phase_id as construction_phase_id,
  a.f_unit_id as third_part_id,
  a.f_unit_pid as third_part_parent_id,
  a.item_id as item_id,
  a.sub_item_id as sub_item_id,
  '0_bim_202005181910_0' as table_type
from
  ods.ods_pbp_segment_group_0_bim_10_0_new a
union all
select
  b.id as id,
  b.project_id as project_id,
  b.full_id as group_full_id,
  b.parent_id as parent_id,
  b.specialty_id as specialty_id,
  b.building_id as building_id,
  b.floor_id as floor_id,
  b.code as structure_code,
  b.name as structure_name,
  b.remark as remark,
  b.type as structure_type,
  b.order_no as order_no,
  b.version as version,
  b.is_delete as is_deleted,
  b.sys_revision as sys_revision,
  b.sys_create_time as sys_create_time,
  b.sys_modified_time as sys_modified_time,
  b.sys_creator_id as sys_creator_id,
  b.sys_modifier_id as sys_modifier_id,
  b.construction_phase_id as construction_phase_id,
  b.f_unit_id as third_part_id,
  b.f_unit_pid as third_part_parent_id,
  b.item_id as item_id,
  b.sub_item_id as sub_item_id,
  '1_bim_202005181910_1' as table_type
from
  ods.ods_pbp_segment_group_1_bim_10_1_new b
union all
select
  c.id as id,
  c.project_id as project_id,
  c.full_id as group_full_id,
  c.parent_id as parent_id,
  c.specialty_id as specialty_id,
  c.building_id as building_id,
  c.floor_id as floor_id,
  c.code as structure_code,
  c.name as structure_name,
  c.remark as remark,
  c.type as structure_type,
  c.order_no as order_no,
  c.version as version,
  c.is_delete as is_deleted,
  c.sys_revision as sys_revision,
  c.sys_create_time as sys_create_time,
  c.sys_modified_time as sys_modified_time,
  c.sys_creator_id as sys_creator_id,
  c.sys_modifier_id as sys_modifier_id,
  c.construction_phase_id as construction_phase_id,
  c.f_unit_id as third_part_id,
  c.f_unit_pid as third_part_parent_id,
  c.item_id as item_id,
  c.sub_item_id as sub_item_id,
  '0_bim_1120655100311625730_0' as table_type
from
  ods.ods_pbp_segment_group_0_bim_30_0_new c
union all
select
  d.id as id,
  d.project_id as project_id,
  d.full_id as group_full_id,
  d.parent_id as parent_id,
  d.specialty_id as specialty_id,
  d.building_id as building_id,
  d.floor_id as floor_id,
  d.code as structure_code,
  d.name as structure_name,
  d.remark as remark,
  d.type as structure_type,
  d.order_no as order_no,
  d.version as version,
  d.is_delete as is_deleted,
  d.sys_revision as sys_revision,
  d.sys_create_time as sys_create_time,
  d.sys_modified_time as sys_modified_time,
  d.sys_creator_id as sys_creator_id,
  d.sys_modifier_id as sys_modifier_id,
  d.construction_phase_id as construction_phase_id,
  d.f_unit_id as third_part_id,
  d.f_unit_pid as third_part_parent_id,
  d.item_id as item_id,
  d.sub_item_id as sub_item_id,
  '1_bim_1120655100311625730_1' as table_type
from
  ods.ods_pbp_segment_group_1_bim_30_1_new d;
  
```

###### ods.ods_pbp_segment_group_0_bim_10_0_new

##### dwd_pbp_pbs_segment_v
生产中台PBS域施工段视图
``` sql
select
  a.id as id,
  a.project_id as project_id,
  a.full_id as segment_full_id,
  a.segment_group_id as segment_group_id,
  a.building_id as building_id,
  a.specialty_id as specialty_id,
  a.code as segment_code,
  a.name as segment_name,
  a.start_floor_id as start_floor_id,
  a.end_floor_id as end_floor_id,
  a.start_elevation as start_elevation,
  a.end_elevation as end_elevation,
  a.position_type as position_type,
  a.dimension_type as dimension_type,
  a.color as color,
  a.line_width as line_width,
  a.remark as remark,
  a.segment_line_type as segment_line_type,
  a.task_status as task_status,
  a.task_deviation as task_deviation,
  a.plan_start_time as plan_start_time,
  a.plan_finish_time as plan_finish_time,
  a.real_start_time as actual_start_time,
  a.real_finis_time as actual_finish_time,
  a.is_show_line as is_show_wireframe,
  a.is_need_relink as is_need_label,
  a.is_related_edo as is_link_edo,
  a.is_related_task as is_link_task,
  a.version as version,
  a.order_no as order_no,
  a.sys_revision as sys_revision,
  a.is_delete as is_deleted,
  a.sys_create_time as sys_create_time,
  a.sys_modified_time as sys_modified_time,
  a.sys_creator_id as sys_creator_id,
  a.sys_modifier_id as sys_modifier_id,
  a.area as segment_area,
  a.area_modification as segment_area_modification,
  a.relate_type as link_type,
  a.construction_phase_id as construction_phase_id,
  a.model_construct_id as model_segment_id,
  a.item_id as item_id,
  a.sub_item_id as sub_item_id,
  '0_bim_202005181910_0' as table_type  
from
  ods.ods_pbp_segment_0_bim_10_0_new a
union all
select
  b.id as id,
  b.project_id as project_id,
  b.full_id as segment_full_id,
  b.segment_group_id as segment_group_id,
  b.building_id as building_id,
  b.specialty_id as specialty_id,
  b.code as segment_code,
  b.name as segment_name,
  b.start_floor_id as start_floor_id,
  b.end_floor_id as end_floor_id,
  b.start_elevation as start_elevation,
  b.end_elevation as end_elevation,
  b.position_type as position_type,
  b.dimension_type as dimension_type,
  b.color as color,
  b.line_width as line_width,
  b.remark as remark,
  b.segment_line_type as segment_line_type,
  b.task_status as task_status,
  b.task_deviation as task_deviation,
  b.plan_start_time as plan_start_time,
  b.plan_finish_time as plan_finish_time,
  b.real_start_time as actual_start_time,
  b.real_finis_time as actual_finish_time,
  b.is_show_line as is_show_wireframe,
  b.is_need_relink as is_need_label,
  b.is_related_edo as is_link_edo,
  b.is_related_task as is_link_task,
  b.version as version,
  b.order_no as order_no,
  b.sys_revision as sys_revision,
  b.is_delete as is_deleted,
  b.sys_create_time as sys_create_time,
  b.sys_modified_time as sys_modified_time,
  b.sys_creator_id as sys_creator_id,
  b.sys_modifier_id as sys_modifier_id,
  b.area as segment_area,
  b.area_modification as segment_area_modification,
  b.relate_type as link_type,
  b.construction_phase_id as construction_phase_id,
  b.model_construct_id as model_segment_id,
  b.item_id as item_id,
  b.sub_item_id as sub_item_id,
  '1_bim_202005181910_1' as table_type  
from
  ods.ods_pbp_segment_1_bim_10_1_new b
union all
select
  c.id as id,
  c.project_id as project_id,
  c.full_id as segment_full_id,
  c.segment_group_id as segment_group_id,
  c.building_id as building_id,
  c.specialty_id as specialty_id,
  c.code as segment_code,
  c.name as segment_name,
  c.start_floor_id as start_floor_id,
  c.end_floor_id as end_floor_id,
  c.start_elevation as start_elevation,
  c.end_elevation as end_elevation,
  c.position_type as position_type,
  c.dimension_type as dimension_type,
  c.color as color,
  c.line_width as line_width,
  c.remark as remark,
  c.segment_line_type as segment_line_type,
  c.task_status as task_status,
  c.task_deviation as task_deviation,
  c.plan_start_time as plan_start_time,
  c.plan_finish_time as plan_finish_time,
  c.real_start_time as actual_start_time,
  c.real_finis_time as actual_finish_time,
  c.is_show_line as is_show_wireframe,
  c.is_need_relink as is_need_label,
  c.is_related_edo as is_link_edo,
  c.is_related_task as is_link_task,
  c.version as version,
  c.order_no as order_no,
  c.sys_revision as sys_revision,
  c.is_delete as is_deleted,
  c.sys_create_time as sys_create_time,
  c.sys_modified_time as sys_modified_time,
  c.sys_creator_id as sys_creator_id,
  c.sys_modifier_id as sys_modifier_id,
  c.area as segment_area,
  c.area_modification as segment_area_modification,
  c.relate_type as link_type,
  c.construction_phase_id as construction_phase_id,
  c.model_construct_id as model_segment_id,
  c.item_id as item_id,
  c.sub_item_id as sub_item_id,
  '0_bim_1120655100311625730_0' as table_type
from
  ods.ods_pbp_segment_0_bim_30_0_new c
union all
select
  d.id as id,
  d.project_id as project_id,
  d.full_id as segment_full_id,
  d.segment_group_id as segment_group_id,
  d.building_id as building_id,
  d.specialty_id as specialty_id,
  d.code as segment_code,
  d.name as segment_name,
  d.start_floor_id as start_floor_id,
  d.end_floor_id as end_floor_id,
  d.start_elevation as start_elevation,
  d.end_elevation as end_elevation,
  d.position_type as position_type,
  d.dimension_type as dimension_type,
  d.color as color,
  d.line_width as line_width,
  d.remark as remark,
  d.segment_line_type as segment_line_type,
  d.task_status as task_status,
  d.task_deviation as task_deviation,
  d.plan_start_time as plan_start_time,
  d.plan_finish_time as plan_finish_time,
  d.real_start_time as actual_start_time,
  d.real_finis_time as actual_finish_time,
  d.is_show_line as is_show_wireframe,
  d.is_need_relink as is_need_label,
  d.is_related_edo as is_link_edo,
  d.is_related_task as is_link_task,
  d.version as version,
  d.order_no as order_no,
  d.sys_revision as sys_revision,
  d.is_delete as is_deleted,
  d.sys_create_time as sys_create_time,
  d.sys_modified_time as sys_modified_time,
  d.sys_creator_id as sys_creator_id,
  d.sys_modifier_id as sys_modifier_id,
  d.area as segment_area,
  d.area_modification as segment_area_modification,
  d.relate_type as link_type,
  d.construction_phase_id as construction_phase_id,
  d.model_construct_id as model_segment_id,
  d.item_id as item_id,
  d.sub_item_id as sub_item_id,
  '1_bim_1120655100311625730_1' as table_type
from
  ods.ods_pbp_segment_1_bim_30_1_new d;
```
###### ods.ods_pbp_segment_0_bim_10_0_new

```
模型施工段表
ods_pbp_model_construction
CREATE TABLE `ods_pbp_model_construction_1_bim_10_1_new` (
  `id` largeint(40) NOT NULL COMMENT "主键ID",
  `project_id` largeint(40) NOT NULL COMMENT "项目ID",
  `name` varchar(1024) NULL COMMENT "施工段名称",
  `building_id` largeint(40) NULL COMMENT "单体ID",
  `floor_id` largeint(40) NULL COMMENT "楼层ID",
  `phase_id` largeint(40) NULL COMMENT "施工阶段ID",
  `phase_name` varchar(1024) NULL COMMENT "施工阶段",
  `deleted` smallint(6) NULL COMMENT "是否已删除（0为未删除，1为已删除）",
  `sys_create_time` datetime NULL COMMENT "创建时间",
  `sys_creator_id` varchar(128) NULL COMMENT "系统字段：创建用户ID",
  `construct_id` largeint(40) NULL COMMENT "施工段编号",
  `show_label` tinyint(4) NULL COMMENT "未归类施工段是否加载",
  `sys_modified_time` datetime NULL COMMENT "系统-记录修改时间",
  `item_id` largeint(40) NULL COMMENT "",
  `sub_item_id` largeint(40) NULL COMMENT ""
) ENGINE=OLAP 
PRIMARY KEY(`id`, `project_id`)
COMMENT "模型施工段"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);

流水段分组表
ods_pbp_segment_group
CREATE TABLE `ods_pbp_segment_group_0_bim_10_0_new` (
  `id` largeint(40) NOT NULL COMMENT "主键",
  `project_id` largeint(40) NOT NULL COMMENT "项目id",
  `full_id` varchar(65530) NULL COMMENT "全路径ID",
  `parent_id` largeint(40) NULL COMMENT "本表外键",
  `specialty_id` largeint(40) NULL COMMENT "专业",
  `building_id` largeint(40) NULL COMMENT "单体",
  `floor_id` largeint(40) NULL COMMENT "楼层",
  `code` varchar(255) NULL COMMENT "编码",
  `name` varchar(255) NULL COMMENT "名称",
  `remark` varchar(65535) NULL COMMENT "备注",
  `type` bigint(20) NULL COMMENT "分组类型( 1=Building 2=Specialty 3=Floor 4=None )",
  `order_no` int(11) NULL COMMENT "序号",
  `version` int(11) NULL COMMENT "版本号",
  `is_delete` tinyint(4) NULL COMMENT "逻辑删除",
  `sys_revision` largeint(40) NULL COMMENT "系统-乐观锁版本号",
  `sys_create_time` datetime NULL COMMENT "系统-记录创建时间",
  `sys_modified_time` datetime NULL COMMENT "系统-记录修改时间",
  `sys_creator_id` varchar(255) NULL COMMENT "系统-创建人",
  `sys_modifier_id` varchar(255) NULL COMMENT "系统-最后修改人",
  `construction_phase_id` bigint(20) NULL COMMENT "施工阶段id",
  `f_unit_id` varchar(100) NULL COMMENT "三方id",
  `f_unit_pid` varchar(100) NULL COMMENT "三方父id",
  `ext_type` bigint(20) NULL COMMENT "分部及子分部类型",
  `item_id` largeint(40) NULL COMMENT "分部id",
  `sub_item_id` largeint(40) NULL COMMENT "子分部id"
) ENGINE=OLAP 
PRIMARY KEY(`id`, `project_id`)
COMMENT "流水段分组"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);

流水段表
ods_pbp_segment
CREATE TABLE `ods_pbp_segment_0_bim_10_0_new` (
  `id` largeint(40) NOT NULL COMMENT "主键",
  `project_id` largeint(40) NOT NULL COMMENT "项目id",
  `full_id` varchar(1000) NULL COMMENT "全路径ID",
  `segment_group_id` largeint(40) NULL COMMENT "流水分组",
  `building_id` largeint(40) NULL COMMENT "单体",
  `specialty_id` largeint(40) NULL COMMENT "专业",
  `code` varchar(255) NULL COMMENT "编码",
  `name` varchar(255) NULL COMMENT "名称",
  `start_floor_id` largeint(40) NULL COMMENT "起始楼层",
  `end_floor_id` largeint(40) NULL COMMENT "终止楼层",
  `start_elevation` decimal(20, 8) NULL COMMENT "起始标高",
  `end_elevation` decimal(20, 8) NULL COMMENT "终止标高",
  `position_type` bigint(20) NULL COMMENT "位置类型( 1=按楼层 2=自定义 )",
  `dimension_type` bigint(20) NULL COMMENT "维度类型( 1=按楼层构件类型 2=按系统 )",
  `color` int(11) NULL COMMENT "颜色",
  `line_width` int(11) NULL COMMENT "线宽",
  `remark` varchar(255) NULL COMMENT "备注",
  `segment_line_type` bigint(20) NULL COMMENT "线框类型( 1=实线 2=点 3=线段 4=点划线 )",
  `task_status` bigint(20) NULL COMMENT "任务状态( 0=未开始 1=进行中 2=已完成 3=未定义1 4=未定义2 5=未定义3 6=未定义4 )",
  `task_deviation` int(11) NULL COMMENT "任务偏差",
  `plan_start_time` datetime NULL COMMENT "计划开始时间",
  `plan_finish_time` datetime NULL COMMENT "计划结束时间",
  `real_start_time` datetime NULL COMMENT "实际开始时间",
  `real_finis_time` datetime NULL COMMENT "实际结束时间",
  `is_show_line` smallint(6) NULL COMMENT "是否显示线框",
  `is_need_relink` smallint(6) NULL COMMENT "需要调整标记",
  `is_related_edo` smallint(6) NULL COMMENT "是否关联图元",
  `is_related_task` smallint(6) NULL COMMENT "是否关联任务",
  `version` int(11) NULL COMMENT "版本号",
  `order_no` int(11) NULL COMMENT "序号",
  `sys_revision` largeint(40) NULL COMMENT "系统-乐观锁版本号",
  `is_delete` tinyint(4) NULL COMMENT "逻辑删除",
  `sys_create_time` datetime NULL COMMENT "系统-记录创建时间",
  `sys_modified_time` datetime NULL COMMENT "系统-记录修改时间",
  `sys_creator_id` varchar(255) NULL COMMENT "系统-创建人",
  `sys_modifier_id` varchar(255) NULL COMMENT "系统-最后修改人",
  `area` double NULL COMMENT "施工段建筑面积",
  `area_modification` double NULL COMMENT "施工段建筑面积修正值",
  `relate_type` bigint(20) NULL COMMENT "关联类型（1=GTJ 2=其他）",
  `construction_phase_id` bigint(20) NULL COMMENT "施工阶段id",
  `model_construct_id` bigint(20) NULL COMMENT "模型施工段id",
  `item_id` largeint(40) NULL COMMENT "",
  `sub_item_id` largeint(40) NULL COMMENT ""
) ENGINE=OLAP 
PRIMARY KEY(`id`, `project_id`)
COMMENT "流水段"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);

项目PBS分部工程量统计接口
select
    q.element_type_id as elementTypeId,
    q.element_type_name as elementTypeName,
    q.quantity_name as quantityName,
    q.quantity_expression as quantityExpression,
    q.measure_unit as measureUnit,
    round ( sum ( case when q.quantity_type_code = 1 then q.quantity_value else 0 end ), 6 ) as budget,
    round ( sum ( case when q.quantity_type_code = 2 then q.quantity_value else 0 end ), 6 ) as deepen ,
    round ( sum ( case when q.quantity_type_code = 4 then q.quantity_value else 0 end ), 6 ) as retrofit,
    round ( sum ( case when q.quantity_type_code = 5 then q.quantity_value else 0 end ), 6 ) as budgetInput,
    round ( sum ( case when q.quantity_type_code = 6 then q.quantity_value else 0 end ), 6 ) as deepenInput,
    round ( sum ( case when q.quantity_type_code = 7 then q.quantity_value else 0 end ), 6 ) as retrofitInput
from dws.dws_pbp_pbs_sub_item_model_quantity_detail_v q
LEFT SEMI JOIN dws.dws_pbp_pbs_sub_item_element_type_detail_v gpe on
  gpe.project_id = q.project_id
  and gpe.element_type_id = q.element_type_id
  and gpe.sub_item_id = q.sub_item_id 
  and gpe.quantity_expression = q.quantity_expression
  and gpe.system_type = q.system_type
where q.project_id = # { projectid } 
  and q.structure_type_name = #{ structureTypeName }
  # [ and q.table_type = ## { tableType } ] #
  # [ and q.pbs_id in ($$ { nodeIds }) ] # 
  # [ and q.element_type_id = ## { elementTypeId } ] #
group by
 q.element_type_id,
 q.element_type_name,
 q.quantity_name,
 q.measure_unit,
 q.quantity_expression
order by
    q.element_type_name,
    q.quantity_name,
    q.quantity_expression

生产中台PBS分部工程量明细视图
dws_pbp_pbs_sub_item_model_quantity_detail_v
select 
  pbs_id,
  full_id,
  project_id,
  sub_special_id,
  quantity_type_code,
  building_id,
  floor_id,
  construction_phase_id,
  segment_id,
  element_id,
  element_name,
  element_type_id,
  element_type_name,
  quantity_expression,
  quantity_name,
  quantity_value,
  measure_unit,
  model_specification,
  sub_item_id,
  structure_type_name,
  system_type,
  table_type
from
  (
    select
      p.id as pbs_id,
      p.full_id,
      q.project_id,
      q.sub_special_id,
      q.quantity_type_code,
      q.building_id,
      q.floor_id,
      q.construction_phase_id,
      q.segment_id,
      q.element_id,
      q.element_name,
      q.element_type_id,
      q.element_type_name,
      q.quantity_expression,
      q.quantity_name,
      q.quantity_value,
      q.measure_unit,
      q.model_specification,
      p.sub_item_id,
      'other' as structure_type_name,
      q.system_type,
      p.table_type
    from
      dwd.dwd_pbp_calculate_element_model_quantity_v q
      join dwd.dwd_pbp_pbs_structure_sub_item_fill_v p on p.project_id = q.project_id
      and p.building_id = q.building_id
      and p.floor_id = q.floor_id
    where
      p.is_deleted = 0
    UNION ALL
    select
      p.id as pbs_id,
      p.full_id,
      q.project_id,
      q.sub_special_id,
      q.quantity_type_code,
      q.building_id,
      q.floor_id,
      q.construction_phase_id,
      q.segment_id,
      q.element_id,
      q.element_name,
      q.element_type_id,
      q.element_type_name,
      q.quantity_expression,
      q.quantity_name,
      q.quantity_value,
      q.measure_unit,
      q.model_specification,
      p.sub_item_id,
      'building' as structure_type_name,
      q.system_type,
      p.table_type
    from
      dwd.dwd_pbp_calculate_element_model_quantity_v q
      join dwd.dwd_pbp_pbs_structure_sub_item_fill_v p on p.project_id = q.project_id
      and p.building_id = q.building_id
    where
      p.is_deleted = 0
  ) total
```

## dws_pbp_pbs_sub_item_element_type_detail_v
生产中台PBS分部构件类型明细视图
``` sql
SELECT DISTINCT
  gpsiall.sub_item_id,
  gpsiall.project_id,
  gpsiet.element_type_id,
  psetc.quantity_code as quantity_expression,
  CASE
    WHEN LENGTH (gpsiet.system_type_code) = 0 THEN -1
    WHEN gpsiet.system_type_code IS NULL THEN -1
    ELSE gpsiet.system_type_code
  END as system_type
FROM
  (
    SELECT
      gpsi.id as sub_item_id,
      gpsi.project_id as project_id,
      gpsi2.id as sub_entry_id
    FROM
      dwd.dwd_pbp_standard_sub_item_v gpsi
      JOIN dwd.dwd_pbp_standard_sub_item_v gpsi2 ON gpsi.project_id = gpsi2.project_id and starts_with(gpsi2.full_id, gpsi.full_id)=1
    WHERE
      (
        gpsi.level_type = 101
        or gpsi.level_type = 103
      )
      and gpsi2.level_type = 102
      and gpsi.is_deleted = 0
      and gpsi2.is_deleted = 0
  ) gpsiall
  LEFT JOIN dwd.dwd_pbp_standard_element_type_quantity_v gpsiet on gpsiall.project_id = gpsiet.project_id and gpsiall.sub_entry_id = gpsiet.sub_item_id and gpsiet.is_deleted = 0
  LEFT JOIN dwd.dwd_pbp_standard_element_type_coefficient_v psetc on cast(gpsiall.project_id as bigint) = psetc.project_id and gpsiet.id = psetc.element_type_quantity_id and psetc.is_deleted = 0
```

生产中台PBS域分解结构填充视图-分部分项
dwd_pbp_pbs_structure_sub_item_fill_v
select 
 s.id,
 s.project_id,
 s.full_id,
 s.parent_id,
 s.specialty_id,
 ifnull(s.building_id,0) as building_id,
 ifnull(s.floor_id,0) as floor_id,
 s.structure_code,
 s.structure_name,
 s.remark,
 s.order_no,
 s.version,
 s.is_deleted,
 s.sys_revision,
 s.sys_create_time,
 s.sys_modified_time,
 s.sys_creator_id,
 s.sys_modifier_id,
 ifnull(s.construction_phase_id,0) as construction_phase_id,
 s.structure_type,
 ifnull(s.model_segment_id,0) as model_segment_id,
 s.table_type,
 CASE
   WHEN s.sub_item_id IS NULL THEN s.item_id
   ELSE s.sub_item_id
 END as sub_item_id
from dwd.dwd_pbp_pbs_structure_sub_item_v s
where s.is_deleted = 0

生产中台PBS域分解结构视图-分部分项
dwd_pbp_pbs_structure_sub_item_v
 select
   a.id as id,
   a.project_id as project_id,
   a.group_full_id as full_id,
   a.parent_id as parent_id,
   a.specialty_id as specialty_id,
   a.building_id as building_id,
   a.floor_id as floor_id,
   a.structure_code as structure_code,
   a.structure_name as structure_name,
   a.remark as remark,
   a.order_no as order_no,
   a.version as version,
   a.is_deleted as is_deleted,
   a.sys_revision as sys_revision,
   a.sys_create_time as sys_create_time,
   a.sys_modified_time as sys_modified_time,
   a.sys_creator_id as sys_creator_id,
   a.sys_modifier_id as sys_modifier_id,
   a.construction_phase_id as construction_phase_id,
   a.structure_type as structure_type,
   null as model_segment_id,
   a.item_id as item_id,
   a.sub_item_id as sub_item_id,
   a.table_type as table_type
 from
   dim.dim_pbp_pbs_segment_group_v a

生产中台标准域分部分项结构视图
### dwd_pbp_standard_sub_item_v
``` sql
select
  a.id as id,
  a.pid as parent_id,
  a.full_id as full_id,
  a.project_id as project_id,
  a.code as sub_item_code,
  a.name as sub_item_name,
  a.level_type as level_type,
  a.sub_order_no as order_no,
  a.unit as measure_unit,
  a.work_type_code as job_type_code,
  a.work_type_id as job_type_id,
  a.divide_explain as divide_desc,
--   a.creator_id as creator_id,
--   a.modify_id as update_id,
  a.deleted as is_deleted,
--   a.create_time as create_time,
--   a.modify_time as update_time,
  a.remark as remark,
  a.source_id as source_id,
  a.source as data_source,
  a.construct_type_id as construct_type_id,
  a.construct_type_code as construct_type_code,
  a.speciality_type_id as specialty_type_id,
  a.speciality_type_code as specialty_type_code,
  a.work_type_name as job_type_name,
  a.work_type_catalog_id as job_type_catalog_id,
  a.unit_code as measure_unit_code,
  a.work_type_catalog_code as job_type_catalog_code,
  a.sys_create_time as sys_create_time,
  a.sys_modified_time as sys_modified_time,
  a.sys_creator_id as sys_creator_id,
  a.sys_modified_id as sys_modified_id
from
  ods.ods_pbp_gpbp_pbs_sub_item_pf a;
```
#### ods.ods_pbp_gpbp_pbs_sub_item_pf


量表模版（工程量算量模版）
ods_pbp_gpbp_pbs_sub_item_pf
CREATE TABLE `ods_pbp_gpbp_pbs_sub_item_pf` (
  `id` bigint(20) NOT NULL COMMENT "分项ID",
  `project_id` largeint(40) NOT NULL COMMENT "项目id",
  `pid` bigint(20) NULL COMMENT "父id",
  `full_id` varchar(512) NULL COMMENT "全路径ID",
  `code` varchar(100) NULL COMMENT "模版编码",
  `name` varchar(100) NULL COMMENT "模板名称",
  `construction_phase_code` varchar(16) NULL COMMENT "施工阶段编码",
  `level_type` int(11) NULL COMMENT "模版层级类型 100:未知 101:分部 102:分项 103:子分部 104:子分项 105:任务",
  `sub_order_no` int(11) NULL COMMENT "节点内排序",
  `unit` varchar(100) NULL COMMENT "计量单位",
  `work_type_code` varchar(100) NULL COMMENT "工种编号",
  `work_type_id` largeint(40) NULL COMMENT "任务关联工种",
  `divide_explain` varchar(255) NULL COMMENT "划分说明",
  `remark` varchar(500) NULL COMMENT "备注",
  `source_id` bigint(20) NULL COMMENT "来源id 0系统 1企业时必填",
  `source` int(11) NULL COMMENT "来源 0系统 1企业 2自建",
  `construct_type_id` bigint(20) NULL COMMENT "行业",
  `construct_type_code` varchar(100) NULL COMMENT "行业code",
  `speciality_type_id` bigint(20) NULL COMMENT "专业",
  `speciality_type_code` varchar(100) NULL COMMENT "专业code",
  `work_type_name` varchar(100) NULL COMMENT "工种类型名称",
  `work_type_catalog_id` bigint(20) NULL COMMENT "工种分组ID",
  `unit_code` varchar(100) NULL COMMENT "计量单位编码",
  `work_type_catalog_code` varchar(100) NULL COMMENT "工种分组code",
  `sys_create_time` datetime NULL COMMENT "系统-记录创建时间",
  `sys_modified_time` datetime NULL COMMENT "系统-记录修改时间",
  `sys_creator_id` varchar(255) NULL COMMENT "系统-创建人",
  `sys_modified_id` varchar(255) NULL COMMENT "系统-最后修改人",
  `assemble_type` smallint(6) NULL COMMENT "装配状态 0:已装配,1:未装配",
  `deleted` smallint(6) NULL COMMENT "",
  INDEX idx_level_type (`level_type`) USING BITMAP
) ENGINE=OLAP 
PRIMARY KEY(`id`, `project_id`)
COMMENT "量表模版（工程量算量模版）_pf"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 
ORDER BY(`id`)
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);


### dwd_pbp_standard_element_type_quantity_v
生产中台标准域构件工程量视图
``` sql
select
  a.id as id,
  a.project_id as project_id,
  a.sub_item_id as sub_item_id,
  a.element_type_id as element_type_id,
  a.element_type_name as element_type_name,
  a.quantity_name as quantity_names,
  a.quantities_expression as quantity_expression,
  a.unit as measure_unit,
  a.deleted as is_deleted,
  -- a.create_time as create_time,
  -- a.modify_time as update_time,
  a.sys_create_time as sys_create_time,
  a.sys_modified_time as sys_modified_time,
  a.sys_creator_id as sys_creator_id,
  a.sys_modified_id as sys_modified_id,
  a.system_type_code as system_type_code,
  a.system_type_name as system_type_name
from
  ods.ods_pbp_gpbp_pbs_sub_item_element_type_pf a
where a.deleted=0;
```
#### ods.ods_pbp_gpbp_pbs_sub_item_element_type_pf

分部分项构件类型（构件等相关配置）表
ods_pbp_gpbp_pbs_sub_item_element_type_pf
CREATE TABLE `ods_pbp_gpbp_pbs_sub_item_element_type_pf` (
  `id` bigint(20) NOT NULL COMMENT "分项构件类型ID",
  `project_id` largeint(40) NOT NULL COMMENT "当前项目ID",
  `sub_item_id` largeint(40) NULL COMMENT "分项ID",
  `element_type_id` bigint(20) NULL COMMENT "构件类型ID",
  `element_type_name` varchar(64) NULL COMMENT "构件类型名称",
  `quantity_name` varchar(512) NULL COMMENT "工程量名称",
  `quantities_expression` varchar(512) NULL COMMENT "工程变量表达式",
  `unit` varchar(20) NULL COMMENT "单位",
  `deleted` smallint(6) NULL COMMENT "",
  `sys_create_time` datetime NULL COMMENT "系统-记录创建时间",
  `sys_modified_time` datetime NULL COMMENT "系统-记录修改时间",
  `sys_creator_id` varchar(255) NULL COMMENT "系统-创建人",
  `sys_modified_id` varchar(255) NULL COMMENT "系统-最后修改人",
  `system_type_code` varchar(255) NULL COMMENT "专业id（安装），目前只是保存，暂时没有地方用（24730）",
  `gqp_specialty_id` bigint(20) NULL COMMENT "映射后的平台系统类型code",
  `system_type_name` varchar(255) NULL COMMENT "系统类型name",
  `element_group_id` bigint(20) NULL COMMENT "构件分组id"
) ENGINE=OLAP 
PRIMARY KEY(`id`, `project_id`)
COMMENT "分部分项构件类型（构件等相关配置）_pf"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 
ORDER BY(`id`)
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);


### dwd_pbp_standard_element_type_coefficient_v
生产中台标准域构件工程量系数视图
``` sql
select
  a.id as id,
  a.project_id as project_id,
  a.sub_item_element_type_id as element_type_quantity_id,
  a.name as quantity_name,
  a.code as quantity_code,
  a.factor as coefficient,
  a.unit as measure_unit,
  a.deleted as is_deleted,
--   a.create_time as create_time,
--   a.modify_time as update_time,
  a.sys_create_time as sys_create_time,
  a.sys_modified_time as sys_modified_time,
  a.sys_creator_id as sys_creator_id,
  a.sys_modified_id as sys_modified_id
from
  ods.ods_pbp_gpbp_pbs_sub_item_element_type_quantity_pf a
where a.deleted=0;
```
#### ods.ods_pbp_gpbp_pbs_sub_item_element_type_quantity_pf

分部分项构件类型工程量（构件等相关配置）表
ods_pbp_gpbp_pbs_sub_item_element_type_quantity_pf
CREATE TABLE `ods_pbp_gpbp_pbs_sub_item_element_type_quantity_pf` (
  `id` bigint(20) NOT NULL COMMENT "ID",
  `project_id` bigint(20) NOT NULL COMMENT "项目id",
  `system_type_name` varchar(255) NULL COMMENT "",
  `sub_item_element_type_id` largeint(40) NULL COMMENT "分项工构件类型表ID",
  `system_type_code` varchar(255) NULL COMMENT "",
  `name` varchar(200) NULL COMMENT "工程量名称",
  `code` varchar(200) NULL COMMENT "工程变量编号",
  `factor` decimal(10, 2) NULL COMMENT "系数",
  `unit` varchar(20) NULL COMMENT "单位",
  `deleted` smallint(6) NULL COMMENT "",
  `sys_create_time` datetime NULL COMMENT "系统-记录创建时间",
  `sys_modified_time` datetime NULL COMMENT "系统-记录修改时间",
  `sys_creator_id` varchar(255) NULL COMMENT "系统-创建人",
  `sys_modified_id` varchar(255) NULL COMMENT "系统-最后修改人",
  `gqp_specialty_id` bigint(20) NULL COMMENT "",
  `element_group_id` bigint(20) NULL COMMENT "构件分組id"
) ENGINE=OLAP 
PRIMARY KEY(`id`, `project_id`)
COMMENT "分部分项构件类型（构件等相关配置）_pf"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 
ORDER BY(`id`)
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);

项目WBS工程量统计接口
/gdmp/projects/%s/wbs_model_quantity_statistic
select
    t.element_type_id as elementTypeId,
    t.element_type_name as elementTypeName,
    t.quantity_name as quantityName,
    t.measure_unit as measureUnit,
    t.quantity_expression as quantityExpression,
    round ( sum ( case when t.quantity_type_code = 1 then t.quantity_value else 0 end ), 6 ) as budget,
    round ( sum ( case when t.quantity_type_code = 2 then t.quantity_value else 0 end ), 6 ) as deepen ,
    round ( sum ( case when t.quantity_type_code = 4 then t.quantity_value else 0 end ), 6 ) as retrofit,
    round ( sum ( case when t.quantity_type_code = 5 then t.quantity_value else 0 end ), 6 ) as budgetInput,
    round ( sum ( case when t.quantity_type_code = 6 then t.quantity_value else 0 end ), 6 ) as deepenInput,
    round ( sum ( case when t.quantity_type_code = 7 then t.quantity_value else 0 end ), 6 ) as retrofitInput
from dws.dws_pbp_wbs_model_quantity_detail_v t
where t.project_id = # { projectid }
    and t.task_id = # { taskId }
    # [ and t.element_type_id = ## { elementTypeId } ] #
group by
    t.element_type_id,
    t.element_type_name,
    t.quantity_name,
    t.measure_unit,
    t.quantity_expression
order by
    t.element_type_name,
    t.quantity_name


## dws_pbp_wbs_model_quantity_detail_v
生产中台WBS工程量明细视图
``` sql
select
    q.project_id,
    q.table_type,
    q.full_id,
    q.pbs_id,
    e.task_id,
    q.element_id,
    q.element_name,
    q.element_type_id,
    q.element_type_name,
    q.quantity_type_code,
    e.quantity_expression,
    q.quantity_name,
    q.measure_unit,
    q.quantity_value * e.coefficient as quantity_value,
    q.model_specification
from dws.dws_pbp_pbs_model_quantity_detail_v as q
join dws.dws_pbp_wbs_quantity_expression_v as e
    on q.project_id = e.project_id
    and q.table_type = e.table_type
    and q.pbs_id = e.pbs_id
    and q.quantity_expression = e.quantity_expression
    and q.element_type_id = e.element_type_id ;
```



### dws_pbp_wbs_quantity_expression_v
生产中台WBS算量表达式视图
``` sql
select
  t.id as task_id,
  t.project_id,
  case
    when t.segment_id is not null then t.segment_id
    else t.segment_group_id
  end as pbs_id,
  t.task_name,
  e.quantity_expression,
  e.element_type_id,
  e.coefficient,
  t.table_type,
  e.system_type
from
  dim.dim_pbp_calculate_quantity_expression_v e
  join dwd.dwd_pbp_calculate_period_task_v t on t.project_id = e.project_id
  and t.table_type = e.table_type
  and t.id = e.task_id
where
  t.is_deleted = 0
  and e.is_deleted = 0;
```



#### dim_pbp_calculate_quantity_expression_v
生产中台工程量表达式维表视图
``` sql
select
  a.id as id,
  a.project_id as project_id,
  a.relate_task_id as task_id,
  a.quantity_expression as quantity_expression,
  a.element_type_id as element_type_id,
  a.coefficient as coefficient,
  a.is_delete as is_deleted,
  a.sys_revision as sys_revision,
  a.sys_create_time as sys_create_time,
  a.sys_modified_time as sys_modified_time,
  a.sys_creator_id as sys_creator_id,
  a.sys_modifier_id as sys_modifier_id,
  '0_bim_202005181910_0' as table_type,
  a.system_type
from
  ods.ods_pbp_element_quantity_exp_0_bim_10_0_new a
union all
select
  b.id as id,
  b.project_id as project_id,
  b.relate_task_id as task_id,
  b.quantity_expression as quantity_expression,
  b.element_type_id as element_type_id,
  b.coefficient as coefficient,
  b.is_delete as is_deleted,
  b.sys_revision as sys_revision,
  b.sys_create_time as sys_create_time,
  b.sys_modified_time as sys_modified_time,
  b.sys_creator_id as sys_creator_id,
  b.sys_modifier_id as sys_modifier_id,
  '1_bim_202005181910_1' as table_type,
  b.system_type
from
  ods.ods_pbp_element_quantity_exp_1_bim_10_1_new b
union all
select
  c.id as id,
  c.project_id as project_id,
  c.relate_task_id as task_id,
  c.quantity_expression as quantity_expression,
  c.element_type_id as element_type_id,
  c.coefficient as coefficient,
  c.is_delete as is_deleted,
  c.sys_revision as sys_revision,
  c.sys_create_time as sys_create_time,
  c.sys_modified_time as sys_modified_time,
  c.sys_creator_id as sys_creator_id,
  c.sys_modifier_id as sys_modifier_id,
  '0_bim_1120655100311625730_0' as table_type,
  c.system_type
from
  ods.ods_pbp_element_quantity_exp_0_bim_30_0_new c
union all
select
  d.id as id,
  d.project_id as project_id,
  d.relate_task_id as task_id,
  d.quantity_expression as quantity_expression,
  d.element_type_id as element_type_id,
  d.coefficient as coefficient,
  d.is_delete as is_deleted,
  d.sys_revision as sys_revision,
  d.sys_create_time as sys_create_time,
  d.sys_modified_time as sys_modified_time,
  d.sys_creator_id as sys_creator_id,
  d.sys_modifier_id as sys_modifier_id,
  '1_bim_1120655100311625730_1' as table_type,
  d.system_type
from
  ods.ods_pbp_element_quantity_exp_1_bim_30_1_new d;
```

##### ods.ods_pbp_element_quantity_exp_0_bim_10_0_new

构件工程量表达式信息表
ods_pbp_element_quantity_exp
CREATE TABLE `ods_pbp_element_quantity_exp_1_bim_30_1_new` (
  `id` largeint(40) NOT NULL COMMENT "id",
  `project_id` largeint(40) NOT NULL COMMENT "项目id",
  `relate_task_id` largeint(40) NULL COMMENT "关联任务id",
  `quantity_expression` varchar(255) NULL COMMENT "工程量表达式",
  `element_type_id` int(11) NULL COMMENT "构件类型",
  `coefficient` decimal(10, 4) NULL COMMENT "系数",
  `is_delete` tinyint(4) NULL COMMENT "逻辑删除",
  `sys_revision` largeint(40) NULL COMMENT "系统-乐观锁版本号",
  `sys_create_time` datetime NULL COMMENT "系统-记录创建时间",
  `sys_modified_time` datetime NULL COMMENT "系统-记录修改时间",
  `sys_creator_id` varchar(255) NULL COMMENT "系统-创建人",
  `sys_modifier_id` varchar(255) NULL COMMENT "系统-最后修改人",
  `system_type` varchar(32) NULL COMMENT ""
) ENGINE=OLAP 
PRIMARY KEY(`id`, `project_id`)
COMMENT "构件工程量表达式信息"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);

生产中台算量域工期任务视图
dwd_pbp_calculate_period_task_v
select
  a.id as id,
  a.project_id as project_id,
  a.segment_group_id as segment_group_id,
  a.segment_id as segment_id,
  a.period_standard_code as standard_period_code,
  a.period_standard_name as standard_period_name,
  a.construction_phase_code as construction_phase_code,
  a.construction_phase_name as construction_phase_name,
  a.task_name as task_name,
  a.measure_unit as measure_unit,
  a.reference_work_period_days as reference_period_days,
  a.work_efficiency as work_efficiency,
  a.work_type_code as job_type_code,
  a.measure_unit_code as measure_unit_code,
  a.type as group_type,
  a.is_delete as is_deleted,
  a.sys_revision as sys_revision,
  a.sys_create_time as sys_create_time,
  a.sys_modified_time as sys_modified_time,
  a.sys_creator_id as sys_creator_id,
  a.sys_modifier_id as sys_modifier_id,
  a.node_type as structure_type,
  a.work_type_name as job_type_name,
  '0_bim_202005181910_0' as table_type
from
  ods.ods_pbp_sub_task_0_bim_10_0_new a
union all
select
  b.id as id,
  b.project_id as project_id,
  b.segment_group_id as segment_group_id,
  b.segment_id as segment_id,
  b.period_standard_code as standard_period_code,
  b.period_standard_name as standard_period_name,
  b.construction_phase_code as construction_phase_code,
  b.construction_phase_name as construction_phase_name,
  b.task_name as task_name,
  b.measure_unit as measure_unit,
  b.reference_work_period_days as reference_period_days,
  b.work_efficiency as work_efficiency,
  b.work_type_code as job_type_code,
  b.measure_unit_code as measure_unit_code,
  b.type as group_type,
  b.is_delete as is_deleted,
  b.sys_revision as sys_revision,
  b.sys_create_time as sys_create_time,
  b.sys_modified_time as sys_modified_time,
  b.sys_creator_id as sys_creator_id,
  b.sys_modifier_id as sys_modifier_id,
  b.node_type as structure_type,
  b.work_type_name as job_type_name,
  '1_bim_202005181910_1' as table_type
from
  ods.ods_pbp_sub_task_1_bim_10_1_new b
union all
select
  c.id as id,
  c.project_id as project_id,
  c.segment_group_id as segment_group_id,
  c.segment_id as segment_id,
  c.period_standard_code as standard_period_code,
  c.period_standard_name as standard_period_name,
  c.construction_phase_code as construction_phase_code,
  c.construction_phase_name as construction_phase_name,
  c.task_name as task_name,
  c.measure_unit as measure_unit,
  c.reference_work_period_days as reference_period_days,
  c.work_efficiency as work_efficiency,
  c.work_type_code as job_type_code,
  c.measure_unit_code as measure_unit_code,
  c.type as group_type,
  c.is_delete as is_deleted,
  c.sys_revision as sys_revision,
  c.sys_create_time as sys_create_time,
  c.sys_modified_time as sys_modified_time,
  c.sys_creator_id as sys_creator_id,
  c.sys_modifier_id as sys_modifier_id,
  c.node_type as structure_type,
  c.work_type_name as job_type_name,
  '0_bim_1120655100311625730_0' as table_type
from
  ods.ods_pbp_sub_task_0_bim_30_0_new c
union all
select
  d.id as id,
  d.project_id as project_id,
  d.segment_group_id as segment_group_id,
  d.segment_id as segment_id,
  d.period_standard_code as standard_period_code,
  d.period_standard_name as standard_period_name,
  d.construction_phase_code as construction_phase_code,
  d.construction_phase_name as construction_phase_name,
  d.task_name as task_name,
  d.measure_unit as measure_unit,
  d.reference_work_period_days as reference_period_days,
  d.work_efficiency as work_efficiency,
  d.work_type_code as job_type_code,
  d.measure_unit_code as measure_unit_code,
  d.type as group_type,
  d.is_delete as is_deleted,
  d.sys_revision as sys_revision,
  d.sys_create_time as sys_create_time,
  d.sys_modified_time as sys_modified_time,
  d.sys_creator_id as sys_creator_id,
  d.sys_modifier_id as sys_modifier_id,
  d.node_type as structure_type,
  d.work_type_name as job_type_name,
  '1_bim_1120655100311625730_1' as table_type
from
  ods.ods_pbp_sub_task_1_bim_30_1_new d;

工期任务表
ods_pbp_sub_task
CREATE TABLE `ods_pbp_sub_task_0_bim_10_0_new` (
  `id` largeint(40) NOT NULL COMMENT "id",
  `project_id` largeint(40) NOT NULL COMMENT "项目id",
  `segment_group_id` largeint(40) NULL COMMENT "部位id",
  `segment_id` largeint(40) NULL COMMENT "施工段部位id",
  `period_standard_code` varchar(32) NULL COMMENT "标准工期包编码",
  `period_standard_name` varchar(255) NULL COMMENT "工期包名称",
  `construction_phase_code` varchar(18) NULL COMMENT "施工阶段编码",
  `construction_phase_name` varchar(255) NULL COMMENT "施工阶段名称",
  `task_name` varchar(255) NULL COMMENT "任务名称",
  `measure_unit` varchar(10) NULL COMMENT "计量单位",
  `reference_work_period_days` decimal(10, 3) NULL COMMENT "参考工期(天)",
  `work_efficiency` decimal(10, 3) NULL COMMENT "工效",
  `work_type_code` varchar(255) NULL COMMENT "工种类型编码",
  `measure_unit_code` varchar(255) NULL COMMENT "计量单位编码",
  `type` bigint(20) NULL COMMENT "分组类型(6=任务)",
  `is_delete` tinyint(4) NULL COMMENT "逻辑删除",
  `sys_revision` largeint(40) NULL COMMENT "系统-乐观锁版本号",
  `sys_create_time` datetime NULL COMMENT "系统-记录创建时间",
  `sys_modified_time` datetime NULL COMMENT "系统-记录修改时间",
  `sys_creator_id` varchar(255) NULL COMMENT "系统-创建人",
  `sys_modifier_id` varchar(255) NULL COMMENT "系统-最后修改人",
  `node_type` int(11) NULL COMMENT "部位类型",
  `work_type_name` varchar(255) NULL COMMENT "工种类型名称",
  `sub_item_id` largeint(40) NULL COMMENT ""
) ENGINE=OLAP 
PRIMARY KEY(`id`, `project_id`)
COMMENT "工期任务表"
DISTRIBUTED BY HASH(`project_id`) BUCKETS 20 
PROPERTIES (
"replication_num" = "3",
"in_memory" = "false",
"storage_format" = "DEFAULT",
"enable_persistent_index" = "true",
"compression" = "LZ4"
);

项目WBS分部工程量统计接口
wbs_sub_item_model_quantity_statistic
select
    t.element_type_id as elementTypeId,
    t.element_type_name as elementTypeName,
    t.quantity_name as quantityName,
    t.measure_unit as measureUnit,
    t.quantity_expression as quantityExpression,
    round ( sum ( case when t.quantity_type_code = 1 then t.quantity_value else 0 end ), 6 ) as budget,
    round ( sum ( case when t.quantity_type_code = 2 then t.quantity_value else 0 end ), 6 ) as deepen ,
    round ( sum ( case when t.quantity_type_code = 4 then t.quantity_value else 0 end ), 6 ) as retrofit,
    round ( sum ( case when t.quantity_type_code = 5 then t.quantity_value else 0 end ), 6 ) as budgetInput,
    round ( sum ( case when t.quantity_type_code = 6 then t.quantity_value else 0 end ), 6 ) as deepenInput,
    round ( sum ( case when t.quantity_type_code = 7 then t.quantity_value else 0 end ), 6 ) as retrofitInput
from dws.dws_pbp_wbs_sub_item_model_quantity_detail_v t
LEFT SEMI JOIN dws.dws_pbp_pbs_sub_item_element_type_detail_v gpe on
  gpe.project_id = t.project_id
  and gpe.element_type_id = t.element_type_id
  and gpe.sub_item_id = t.sub_item_id 
  and gpe.quantity_expression = t.quantity_expression
  and gpe.system_type = t.system_type
where t.project_id = # { projectid }
    and t.structure_type_name = #{ structureTypeName }
    and t.task_id = # { taskId }
    # [ and t.table_type = ## { tableType } ] #
    # [ and t.element_type_id = ## { elementTypeId } ] #
group by
    t.element_type_id,
    t.element_type_name,
    t.quantity_name,
    t.measure_unit,
    t.quantity_expression
order by
    t.element_type_name,
    t.quantity_name,
    t.quantity_expression

生产中台WBS分部工程量明细视图
dws_pbp_wbs_sub_item_model_quantity_detail_v
select
    q.project_id,
    q.full_id,
    q.pbs_id,
    e.task_id,
    q.element_id,
    q.element_name,
    q.element_type_id,
    q.element_type_name,
    q.quantity_type_code,
    e.quantity_expression,
    q.quantity_name,
    q.measure_unit,
    q.quantity_value * e.coefficient as quantity_value,
    q.model_specification,
    q.sub_item_id,
    q.structure_type_name,
    q.system_type,
    q.table_type
from dws.dws_pbp_pbs_sub_item_model_quantity_detail_v as q
join dws.dws_pbp_wbs_quantity_expression_v as e
    on q.project_id = e.project_id
    and q.pbs_id = e.pbs_id
    and q.quantity_expression = e.quantity_expression
    and q.element_type_id = e.element_type_id
    and q.table_type = e.table_type ;


# 材料相关视图
# dws_pbp_pbs_material_quantity_detail_v
```sql
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
```

###  dwd_pbp_calculate_element_type_material_quantity_v
``` sql
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
```

#### `gtcp_quantity`.`gtc_f_element_type_material_quantity`

# dws_pbp_wbs_material_quantity_detail_v
``` sql
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
```

## `dws`.`dws_pbp_wbs_quantity_expression_v`
``` sql
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
```

### `dim`.`dim_pbp_calculate_quantity_expression_v`
``` sql
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
UNION
ALL
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
UNION
ALL
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
UNION
ALL
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
UNION
ALL
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
UNION
ALL
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
UNION
ALL
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
UNION
ALL
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
```
#### `ods`.`ods_pbp_element_quantity_exp_0_bim_10_0_new`
#### `ods`.`ods_pbp_element_quantity_exp_3_bim_30_1_new`


### `dwd`.`dwd_pbp_calculate_period_task_v`
``` sql
CREATE VIEW `dwd_pbp_calculate_period_task_v` (
    `id` COMMENT "主键ID",
    `project_id` COMMENT "项目ID",
    `segment_group_id` COMMENT "施工段分组ID",
    `segment_id` COMMENT "施工段ID",
    `standard_period_code` COMMENT "标准工期包编码",
    `standard_period_name` COMMENT "标准工期包名称",
    `construction_phase_code` COMMENT "施工阶段编码",
    `construction_phase_name` COMMENT "施工阶段名称",
    `task_name` COMMENT "任务名称",
    `measure_unit` COMMENT "计量单位",
    `reference_period_days` COMMENT "参考工期",
    `work_efficiency` COMMENT "工效",
    `job_type_code` COMMENT "工种编码",
    `measure_unit_code` COMMENT "计量单位编码",
    `group_type` COMMENT "分组类型",
    `is_deleted` COMMENT "是否删除",
    `sys_revision` COMMENT "系统-乐观锁版本号",
    `sys_create_time` COMMENT "系统-记录创建时间",
    `sys_modified_time` COMMENT "系统-记录修改时间",
    `sys_creator_id` COMMENT "系统-创建人",
    `sys_modifier_id` COMMENT "系统-最后修改人",
    `structure_type` COMMENT "结构类型",
    `job_type_name` COMMENT "工种名称",
    `table_type` COMMENT "数据所属表编码"
) AS
SELECT
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`id`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`project_id`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`segment_group_id`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`segment_id`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`period_standard_code` AS `standard_period_code`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`period_standard_name` AS `standard_period_name`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`construction_phase_code`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`construction_phase_name`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`task_name`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`measure_unit`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`reference_work_period_days` AS `reference_period_days`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`work_efficiency`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`work_type_code` AS `job_type_code`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`measure_unit_code`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`type` AS `group_type`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`is_delete` AS `is_deleted`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`sys_revision`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`sys_create_time`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`sys_modified_time`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`sys_creator_id`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`sys_modifier_id`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`node_type` AS `structure_type`,
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`.`work_type_name` AS `job_type_name`,
    '0_bim_202005181910_0' AS `table_type`
FROM
    `ods`.`ods_pbp_sub_task_0_bim_10_0_new`
UNION
ALL
SELECT
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`id`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`project_id`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`segment_group_id`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`segment_id`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`period_standard_code` AS `standard_period_code`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`period_standard_name` AS `standard_period_name`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`construction_phase_code`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`construction_phase_name`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`task_name`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`measure_unit`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`reference_work_period_days` AS `reference_period_days`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`work_efficiency`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`work_type_code` AS `job_type_code`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`measure_unit_code`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`type` AS `group_type`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`is_delete` AS `is_deleted`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`sys_revision`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`sys_create_time`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`sys_modified_time`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`sys_creator_id`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`sys_modifier_id`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`node_type` AS `structure_type`,
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`.`work_type_name` AS `job_type_name`,
    '1_bim_202005181910_0' AS `table_type`
FROM
    `ods`.`ods_pbp_sub_task_1_bim_10_0_new`
UNION
ALL
SELECT
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`id`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`project_id`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`segment_group_id`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`segment_id`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`period_standard_code` AS `standard_period_code`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`period_standard_name` AS `standard_period_name`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`construction_phase_code`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`construction_phase_name`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`task_name`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`measure_unit`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`reference_work_period_days` AS `reference_period_days`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`work_efficiency`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`work_type_code` AS `job_type_code`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`measure_unit_code`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`type` AS `group_type`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`is_delete` AS `is_deleted`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`sys_revision`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`sys_create_time`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`sys_modified_time`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`sys_creator_id`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`sys_modifier_id`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`node_type` AS `structure_type`,
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`.`work_type_name` AS `job_type_name`,
    '2_bim_202005181910_1' AS `table_type`
FROM
    `ods`.`ods_pbp_sub_task_2_bim_10_1_new`
UNION
ALL
SELECT
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`id`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`project_id`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`segment_group_id`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`segment_id`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`period_standard_code` AS `standard_period_code`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`period_standard_name` AS `standard_period_name`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`construction_phase_code`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`construction_phase_name`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`task_name`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`measure_unit`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`reference_work_period_days` AS `reference_period_days`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`work_efficiency`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`work_type_code` AS `job_type_code`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`measure_unit_code`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`type` AS `group_type`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`is_delete` AS `is_deleted`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`sys_revision`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`sys_create_time`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`sys_modified_time`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`sys_creator_id`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`sys_modifier_id`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`node_type` AS `structure_type`,
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`.`work_type_name` AS `job_type_name`,
    '3_bim_202005181910_1' AS `table_type`
FROM
    `ods`.`ods_pbp_sub_task_3_bim_10_1_new`
UNION
ALL
SELECT
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`id`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`project_id`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`segment_group_id`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`segment_id`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`period_standard_code` AS `standard_period_code`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`period_standard_name` AS `standard_period_name`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`construction_phase_code`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`construction_phase_name`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`task_name`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`measure_unit`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`reference_work_period_days` AS `reference_period_days`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`work_efficiency`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`work_type_code` AS `job_type_code`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`measure_unit_code`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`type` AS `group_type`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`is_delete` AS `is_deleted`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`sys_revision`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`sys_create_time`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`sys_modified_time`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`sys_creator_id`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`sys_modifier_id`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`node_type` AS `structure_type`,
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`.`work_type_name` AS `job_type_name`,
    '0_bim_1120655100311625730_0' AS `table_type`
FROM
    `ods`.`ods_pbp_sub_task_0_bim_30_0_new`
UNION
ALL
SELECT
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`id`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`project_id`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`segment_group_id`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`segment_id`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`period_standard_code` AS `standard_period_code`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`period_standard_name` AS `standard_period_name`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`construction_phase_code`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`construction_phase_name`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`task_name`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`measure_unit`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`reference_work_period_days` AS `reference_period_days`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`work_efficiency`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`work_type_code` AS `job_type_code`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`measure_unit_code`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`type` AS `group_type`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`is_delete` AS `is_deleted`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`sys_revision`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`sys_create_time`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`sys_modified_time`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`sys_creator_id`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`sys_modifier_id`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`node_type` AS `structure_type`,
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`.`work_type_name` AS `job_type_name`,
    '1_bim_1120655100311625730_0' AS `table_type`
FROM
    `ods`.`ods_pbp_sub_task_1_bim_30_0_new`
UNION
ALL
SELECT
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`id`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`project_id`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`segment_group_id`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`segment_id`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`period_standard_code` AS `standard_period_code`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`period_standard_name` AS `standard_period_name`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`construction_phase_code`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`construction_phase_name`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`task_name`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`measure_unit`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`reference_work_period_days` AS `reference_period_days`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`work_efficiency`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`work_type_code` AS `job_type_code`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`measure_unit_code`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`type` AS `group_type`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`is_delete` AS `is_deleted`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`sys_revision`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`sys_create_time`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`sys_modified_time`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`sys_creator_id`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`sys_modifier_id`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`node_type` AS `structure_type`,
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`.`work_type_name` AS `job_type_name`,
    '2_bim_1120655100311625730_1' AS `table_type`
FROM
    `ods`.`ods_pbp_sub_task_2_bim_30_1_new`
UNION
ALL
SELECT
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`id`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`project_id`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`segment_group_id`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`segment_id`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`period_standard_code` AS `standard_period_code`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`period_standard_name` AS `standard_period_name`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`construction_phase_code`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`construction_phase_name`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`task_name`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`measure_unit`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`reference_work_period_days` AS `reference_period_days`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`work_efficiency`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`work_type_code` AS `job_type_code`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`measure_unit_code`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`type` AS `group_type`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`is_delete` AS `is_deleted`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`sys_revision`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`sys_create_time`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`sys_modified_time`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`sys_creator_id`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`sys_modifier_id`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`node_type` AS `structure_type`,
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`.`work_type_name` AS `job_type_name`,
    '3_bim_1120655100311625730_1' AS `table_type`
FROM
    `ods`.`ods_pbp_sub_task_3_bim_30_1_new`;
```

#### `ods`.`ods_pbp_sub_task_0_bim_10_0_new`
#### `ods`.`ods_pbp_sub_task_3_bim_30_1_new`
# `dws_pbp_wbs_sub_item_material_quantity_detail_v`
``` sql
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
```

## `dws`.`dws_pbp_pbs_sub_item_material_quantity_detail_v`
``` sql
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
        UNION
        ALL
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
```

### `dwd`.`dwd_pbp_calculate_element_type_material_quantity_v`
``` sql
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
```

#### `gtcp_quantity`.`gtc_f_element_type_material_quantity`

## `dws`.`dws_pbp_wbs_quantity_expression_v`