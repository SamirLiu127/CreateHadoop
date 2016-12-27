####tips
```
ZooKeeper 使用2n+1法則

n=1 (保證一台 NameNode Running)

zookeeper = 2 x 1 + 1 = 3

vim  (:set nu 秀行號)
```
#Hadoop Ha
### 1.設定hdfs-site.xml 
※qjournal機器一,二台為NameNode,第三台建議為ResourceNode其中一台
```xml
<property>
  <name>dfs.nameservices</name>
  <value>hadoop-ha</value> <!--NameNode的統稱,可自行命名,之後設定會影響到-->
</property>
<property>
  <name>dfs.ha.namenodes.hadoop-ha</name>
  <value>nn1,nn2</value>
</property>
<property>
  <name>dfs.namenode.rpc-address.hadoop-ha.nn1</name>
  <value>master1.example.org:8020</value>
</property>
<property>
  <name>dfs.namenode.http-address.hadoop-ha.nn1</name>
  <value>master1.example.org:50070</value>
</property>
<property>
  <name>dfs.namenode.rpc-address.hadoop-ha.nn2</name>
  <value>master2.example.org:8020</value>
</property>
<property>
  <name>dfs.namenode.http-address.hadoop-ha.nn2</name>
  <value>master2.example.org:50070</value>
</property>
<property>
  <name>dfs.namenode.shared.edits.dir</name>
  <value>qjournal://master1.example.org:8485;master2.example.org:8485;master3.example.org:8485/hadoop-ha</value> <!--設定QJM:儲存NameNode的資料-->
</property>
<property>
  <name>dfs.journalnode.edits.dir</name>
  <value>/home/hadoop/journalnode</value>   <!--資料儲存的路徑-->
</property>
<property>
  <name>dfs.ha.fencing.methods</name>   <!--避免同時啟動NameNode(腦裂)-->
  <value>sshfence</value>
</property>
<property>
  <name>dfs.ha.fencing.ssh.private-key-files</name>
  <value>/home/hadoop/.ssh/id_rsa</value> #使用rsa_key
</property>

<property> <!--每台,為了角色可便利更換，不是偷懶-->
  <name>dfs.client.failover.proxy.provider.hadoop-ha</name>
  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
```
