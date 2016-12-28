####tips
vim
```vim 
i (insert 開始編輯,Esc離開編輯)
:set nu (秀行號)
:w (儲存)
:q (離開vim)
:wq (儲存後離開)
:q! (不儲存離開)
/word (搜尋,分大小寫)
n (重複上一動,順向,向下搜尋)
N (重複上一動,逆向,向上搜尋)
c (復原前一個動作)
num (num為數字,跳到num行數)
gg  (回到第一行)
```
other
```
scp hdfs-site.xml master2.example.org:/usr/local/hadoop/etc/hadoop (拷貝本地檔案至指定機器指定位置)
```
# Hadoop Ha
### 1.設定hdfs-site.xml 
※qjournal(QJM)機器一,二台為NameNode,第三台建議為ResourceNode其中一台
```xml
<property>
  <name>dfs.nameservices</name>
  <value>hadoop-ha</value> <!--NameNode的變數(統稱),可自行命名,之後設定會影響到-->
</property>
<property>
  <name>dfs.ha.namenodes.hadoop-ha</name>
  <value>nn1,nn2</value> <!--設定變數-->
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
  <value>/home/hadoop/.ssh/id_rsa</value> <!--使用rsa_key-->
</property>

<property> <!--每台,為了角色可便利更換，不是偷懶-->
  <name>dfs.client.failover.proxy.provider.hadoop-ha</name>
  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
```
### 2.建立journalnode目錄(QJM三台)
##### 存放NameNode異動資料
```
> mkdir ~/journalnode
```
### 3.修改core-site.xml
```xml
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://hadoop-ha</value> <!--更改指定NameNode,hadoop-ha為hdfs-site.xml設定值-->
</property>
```
註:若機器沒關,需關掉
### 4.copy NameNode日誌
##### 分別啟動journalnode(QJM三台)
`hadoop-daemon.sh start journalnode`<br>
##### Reload日誌(第一台NameNode)
```
(Format NameNode hdfs - only once, if it's a new cluster)(建立整個新的日誌功能)
hdfs namenode -format
   
(If NameNode hdfs already formatted)(重啟日誌功能)
hdfs namenode -initializeSharedEdits
   
(Start nn1 NameNode)(單獨啟動NameNode,寫入日誌)  
hadoop-daemon.sh start namenode
```
##### copy日誌(第二台NameNode)
```
(Copy the contents of Active NameNode metadata)(拷貝日誌同步)
hdfs namenode -bootstrapStandby
   
(Start nn2 NameNode)(啟動第二台NameNode)
hadoop-daemon.sh start namenode
```
### 5.NameNode重啟 確認
```
> stop-dfs.sh 停止
> start-dfs.sh啟動

查看NameNode狀態(一台active,一台standby)
> hdfs haadmin -getServiceState nn1
> hdfs haadmin -getServiceState nn2

測試(active的NameNode)
hadoop-deamon.sh stop namenode
查看另一台是否active
```
# ZooKeeper(version:3.4.9)
2n+1法則:確保running台數n,求出啟動zookeeper台數<br>
ex.1台NameNode Running, 需啟動3台zookeeper<br>
3台zookeeper:建議與QJM相同,不用出網路即可抓到日誌，效能提升<br>
### 1.安裝 (解壓,更名,改權限)
C:\Users\Student\hadoop_classroom => /vagrant<br>
```
> sudo tar -xvf zookeeper-3.4.9.tar.gz -C /usr/local
> sudo mv /usr/local/zookeeper-3.4.9 /usr/local/zookeeper
> sudo chown -R hadoop:hadoop /usr/local/zookeeper
```
### 2.修改zoo.cfg, 建目錄
```
> cd /usr/local/zookeeper/conf 
> cp zoo_sample.cfg zoo.cfg
> vim zoo.cfg
>> tickTime=2000
   initLimit=10
   syncLimit=5
   dataDir=/usr/local/zookeeper/zoodata
   dataLogDir=/usr/local/zookeeper/logs
   clientPort=2181
   
   server.1=master1.example.org:2888:3888
   server.2=master2.example.org:2888:3888
   server.3=master3.example.org:2888:3888
```
Create "zoodata" directory and a file named 'myid' in it
```
> cd /usr/local/zookeeper
> mkdir logs
> mkdir zoodata

> echo "1" > zoodata/myid (on master1)
> echo "2" > zoodata/myid (on master2)
> echo "3" > zoodata/myid (on master3)
```
### 3.建PATH,ZOOKEEPER_HOME, 載入環境變數
`> vim ~/.bashrc`
```bash
export ZOOKEEPER_HOME=/usr/local/zookeeper 
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```
`> sourse ~/.bashrc`
### 4.啟動zookeeper
`zkServer.sh start`<br>
`jps`QuorumPeerMain
# 
