~~~sql
CREATE EXTERNAL CATALOG `icebug`
comment "Iceberg"
PROPERTIES (
    "type"  =  "iceberg",
    "iceberg.catalog.type"  =  "rest",
    "iceberg.catalog.uri"  =  "http://10.xxx:30901/iceberg",
    "iceberg.catalog.warehouse"  =  "s3://xxxx",
    "aws.s3.access_key"  =  "xxxx",
    "aws.s3.secret_key"  =  "xxxx",
    "aws.s3.enable_ssl"  =  "false",
    "aws.s3.endpoint"  =  "https://obs.cn-north-4.myhuaweicloud.com",
    "aws.s3.enable_path_style_access"  =  "true",
    "aws.s3.region"  =  "cn-north-4",
    "s3.access-key-id"  =  "xx",
    "s3.secret-access-key"  =  "xxxxx",
    "s3.endpoint"  =  "https://obs.cn-north-4000.myhuaweicloud.com"
);
~~~
