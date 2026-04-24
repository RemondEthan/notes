>在sr3.3中，be的启动脚本里，JAVA_OPTS_FOR_JDK9_AND_LATE会覆盖到JAVA_OPTS中，是通过jdk版本来判断的，3.5中做了修改

~~~bash
##3.3
# Appending the option to avoid "process heaper" stack overflow exceptions.
# Tried to adding this option to LIBHDFS_OPTS only, but that doesn't work.
export JAVA_OPTS="$JAVA_OPTS -Djdk.lang.processReaperUseDefaultStackSize=true"
export JAVA_OPTS_FOR_JDK_9_AND_LATER="$JAVA_OPTS_FOR_JDK_9_AND_LATER -Djdk.lang.processReaperUseDefaultStackSize=true"

# check java version and choose correct JAVA_OPTS
JAVA_VERSION=$(jdk_version)
final_java_opt=$JAVA_OPTS
if [[ "$JAVA_VERSION" -gt 8 ]]; then
    if [ -n "$JAVA_OPTS_FOR_JDK_9_AND_LATER" ]; then
        final_java_opt=$JAVA_OPTS_FOR_JDK_9_AND_LATER
    fi
fi
~~~

~~~bash
##3.5
export STARROCKS_HOME=`cd "$curdir/.."; pwd`
source $STARROCKS_HOME/bin/common.sh

export_shared_envvars

check_and_update_max_processes

if [ ${RUN_BE} -eq 1 ] ; then
    export_env_from_conf $STARROCKS_HOME/conf/be.conf
fi
if [ ${RUN_CN} -eq 1 ]; then
    export_env_from_conf $STARROCKS_HOME/conf/cn.conf
fi

if [ $? -ne 0 ]; then
    exit 1
fi

final_java_opt=${JAVA_OPTS}
# Compatible with scenarios upgraded from jdk9~jdk16
if [ ! -z "${JAVA_OPTS_FOR_JDK_9_AND_LATER}" ] ; then
    echo "Warning: Configuration parameter JAVA_OPTS_FOR_JDK_9_AND_LATER is not supported, JAVA_OPTS is the only place to set jvm parameters"
    final_java_opt=${JAVA_OPTS_FOR_JDK_9_AND_LATER}
fi

# Appending the option to avoid "process heaper" stack overflow exceptions.
final_java_opt="$final_java_opt -Djdk.lang.processReaperUseDefaultStackSize=true"
export LIBHDFS_OPTS=$final_java_opt
# Prevent JVM from handling any internally or externally generated signals.
# Otherwise, JVM will overwrite the signal handlers for SIGINT and SIGTERM.
export LIBHDFS_OPTS="$LIBHDFS_OPTS -Xrs"

# HADOOP_CLASSPATH defined in $STARROCKS_HOME/conf/hadoop_env.sh
# put $STARROCKS_HOME/conf ahead of $HADOOP_CLASSPATH so that custom config can replace the config in $HADOOP_CLASSPATH
export CLASSPATH=${STARROCKS_HOME}/lib/jni-packages/starrocks-hadoop-ext.jar:$STARROCKS_HOME/conf:$STARROCKS_HOME/lib/jni-packages/*:$HADOOP_CLASSPATH:$CLASSPATH
~~~
所以，这里两个版本的改动是不一样的，感觉3.5的实现更好
>在看看`export_env_from_conf`的实现,如果最后一行没有回车会把最后一行干掉，holy shit！
~~~bash
export_env_from_conf() {
    while read line; do
    ¦   envline=`echo $line | sed 's/[[:blank:]]*=[[:blank:]]*/=/g' | sed 's/^[[:blank:]]*//g' | egrep "^[[:upper:]]([[:upper:]]|_|[[:digit:]])*="`
    ¦   envline=`eval "echo $envline"`
    ¦   if [[ $envline == *"="* ]]; then
    ¦   ¦   eval 'export "$envline"'
    ¦   fi
    done < $1
}
~~~