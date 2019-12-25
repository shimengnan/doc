# 启动脚本-start.sh

```bash
#!/bin/sh
echo Starting .........

export LANG="en_US.UTF-8"

JAR_NAME="/app/jar/ROOT/SC.jar"


LOGS_DIR="/app/jar/logs/aaaaaa"

mkdir -p $LOGS_DIR/console/
mkdir -p $LOGS_DIR/java_heapdump/
mkdir -p $LOGS_DIR/event

ln -s $LOGS_DIR/event /app/jar/event

STDOUT_FILE=$LOGS_DIR/console/console.log

JAVA_OPTS="-Duser.timezone=GMT+8 -server -Xms3024m -Xmx3024m -XX:PermSize=512m -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=512m "

JAVA_GC=" -XX:+PrintGC -XX:+PrintGCTimeStamps  -XX:+PrintGCDetails  -XX:+PrintGCApplicationStoppedTime -Xloggc:gc.log "

JAVA_USER=" -XX:+UseConcMarkSweepGC "

JAVA_OOM=" -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=$LOGS_DIR/java_heapdump.hprof "

JAVA_CMS=" -XX:+CMSClassUnloadingEnabled -XX:+CMSPermGenSweepingEnabled "

java $JAVA_OPTS $JAVA_GC $JAVA_USER $JAVA_OOM -Dfile.encoding=utf-8 $JAVA_CMS -jar $JAR_NAME >$STDOUT_FILE 2>&1
```

