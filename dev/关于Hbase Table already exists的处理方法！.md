# 关于Hbase Table already exists的处理方法~！

2014年10月20日 14:14:14 [jncqlc](https://me.csdn.net/jncqlc) 阅读数：5496 标签： [hbase](http://so.csdn.net/so/search/s.do?q=hbase&t=blog)[table already exist](http://so.csdn.net/so/search/s.do?q=table%20already%20exist&t=blog)



最近测试Hadoop和Hbase集群，一次断电之后，Hbase无法正常启动，经过一系列配置之后，终于使得Hbase重新启动，但是却出现了新的问题，新建表时，总是提示*Table already exists，很是郁闷，不知道什么原因，虽说以前建过同名的表，可是HDFS上和Hbase相关的东西都已经删除了。最后通过Google找到了解决方案，有可能是zookeeper的原因导致，进入HMaster节点，执行，bin/zkCli.sh* 

*ls /hbase/table,查看是否有要新建的表面，如果有使用rmr命令删除，之后重启Hbase，使用create即可成功。*