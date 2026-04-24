#### 1.环境信息
~~~OBS
export HADOOP_CONF_DIR=;
export HADOOP_CLASSPATH=;
export AWS_REGION=cn-north-4;
export AWS_ACCESS_KEY_ID=HPUA8AQHKYFVIRY2K9EK;
export AWS_SECRET_ACCESS_KEY=mfSrkuHgYvL7J7qN6Yvl5jxXpN7i7k69yKQQbubX;
export AWS_ENDPOINT=obs.cn-north-4.myhuaweicloud.com;
~~~
~~~ bash
export AWS_REGION=cn-north-4;
export AWS_ACCESS_KEY_ID=HPUA8AQHKYFVIRY2K9EK;
export AWS_SECRET_ACCESS_KEY=mfSrkuHgYvL7J7qN6Yvl5jxXpN7i7k69yKQQbubX;
export AWS_ENDPOINT=obs.cn-north-4.myhuaweicloud.com;
#http://10.125.167.209:30912/iceberg
spark-sql \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.type=rest \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.uri=http://gdmp-iceberg-test.glodon.com/iceberg \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.warehouse=s3://gde-data-test/biz-model-data/ \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.io-impl=org.apache.iceberg.aws.s3.S3FileIO \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.s3.access.key=HPUA8AQHKYFVIRY2K9EK \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.s3.secret.key=mfSrkuHgYvL7J7qN6Yvl5jxXpN7i7k69yKQQbubX \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.s3.endpoint=https://obs.cn-north-4.myhuaweicloud.com \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.s3.path.style.access=true \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.prefix=paimon_iceberg_compaction_gde \
  --conf spark.sql.catalog.paimon_iceberg_compaction_gde.credential=xxx \
  --exclude-packages org.apache.httpcomponents:httpclient:4.5.14 \
  --jars /data/install/spark-3.5.2-bin-hadoop3-scala2.13/jars/iceberg-spark-runtime-3.5_2.12-1.8.1.jar,/data/install/spark-3.5.2-bin-hadoop3-scala2.13/jars/iceberg-aws-bundle-1.8.1.jar,/data/install/spark-3.5.2-bin-hadoop3-scala2.13/jars/iceberg-core-1.8.1.jar,/data/install/spark-3.5.2-bin-hadoop3-scala2.13/jars/iceberg-common-1.8.1.jar
~~~

~~~bash
#### run jmeter loop one by one
source ${HOME}/.bashrc
WORK_DIR=$(dirname $0)
pushd $WORK_DIR
step=$(expr $(cat .state) + 1)
mkdir -p result
echo "#####################################Time: $(date) Jmeter job Starting...####################################"
echo "##################################################################################################################################"
echo $step > .state
 cat test.list | while read current_test_name
 do     
	if [[ "$last_test_name" != "" ]];then         
		 echo "reset test case ${last_test_name} ...";
		 sed -i "/testname=\"${last_test_name}\"/ s/true/false/" gde_plan.jmx
	fi
	echo "#########################enable test case ${current_test_name} and run test ####################################";     
	sed -i "/testname=\"${current_test_name}\"/ s/false/true/" gde_plan.jmx
	jmeter -JXms=11G -JXmx=11G -n -t TestPlan.jmx -l result/result_${current_test_name}_${step}.jtl -j result/test_${current_test_name}_${step}.log
	last_test_name=$current_test_name; 
done
echo "##################################################################################################################################"
echo "#####################################Time: $(date) Jmeter job End...#########################################\n"
popd
~~~

~~~sql
SELECT * 
        FROM paimon_iceberg_compaction_gde.gde_material_iceberg_compact_partition_01.gde_t_dic_item_material  
        WHERE f_timestamp <= '2025-08-27 17:21:41' 
          AND f_tenant_id = 1001270121443840
        ORDER BY f_id
        LIMIT 10000 OFFSET 100000;
~~~