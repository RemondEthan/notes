```
spark.sql.catalog.cde_new_ae=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.cde_new_ae.type=rest
spark.sql.catalog.cde_new_ae.uri=http://gdmp-iceberg-test.glodon.com/iceberg
spark.sql.catalog.cde_new_ae.io-impl=org.apache.iceberg.aws.s3.S3FileIO
#spark.sql.catalog.cde_new_ae.warehouse=s3://gtcp-lake-test/cde_new/
spark.sql.catalog.cde_new_ae.warehouse=cde_new
spark.sql.catalog.cde_new_ae.s3.endpoint=https://obs.cn-north-4.myhuaweicloud.com
spark.sql.catalog.cde_new_ae.s3.access-key-id=HPUA5LPHUILEEXPKT8OH
spark.sql.catalog.cde_new_ae.s3.secret-access-key=KVo8imHvIGzB4NxLRmwTWsk0zMhb2ztkVYeSVyVm
spark.sql.catalog.cde_new_ae.cache-enabled=false
spark.sql.catalog.cde_new_ae.prefix=cde_new
spark.sql.catalog.cde_new_ae.credential=xx
```




VCPKG_TOOLCHAIN_PATH='/data/duckdb-iceberg/vcpkg/scripts/buildsystems/vcpkg.cmake' make