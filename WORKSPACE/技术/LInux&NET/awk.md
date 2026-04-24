~~~shell
grep bim sr_tabels.csv|head -1|awk -F, '{printf($1",");split($1,parts_1,"_");database_prefix=parts_1[1];split($2,parts_2,"_");table_suffix="";for(i=3;i<=length(parts_2)&&parts_2[i]!=database_prefix;i++){table_suffix=table_suffix"_"parts_2[i]};printf("ods_"$1""table_suffix"\n")}'
~~~
