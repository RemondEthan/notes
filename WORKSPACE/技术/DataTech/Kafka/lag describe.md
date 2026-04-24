~~~shell
kafka-consumer-groups.sh --bootstrap-server 10.125.167.72:9092 --list|while read group
do     
	topic_and_lag=`kafka-consumer-groups.sh --bootstrap-server 10.125.167.72:9092 --group ${group} --describe|awk '{lag+=$6}END{print $2","lag}'`
	echo "${group},${topic_and_lag}"|awk -F, '{if($3>100){print $0}}'
done 2>/dev/null
~~~
