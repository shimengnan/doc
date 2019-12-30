```shell
curl "http://skyworld-eureka-one.jishu.idc:19002/gantry/setOfflineService?ip=$MY_POD_IP" -o /viewlogs/console/curl.log ; sleep 40 ; killall -SIGTERM java > /viewlogs/console/kill.log 2>&1
```

```java
curl "http://10.143.131.134:19002/gantry/setOfflineService?ip=$MY_POD_IP" -o /viewlogs/console/curl.log ; sleep 40 ; killall -SIGTERM java > /viewlogs/console/kill.log 2>&1
```


