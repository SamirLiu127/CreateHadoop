### Hadoop
NameNode(HA) :主要負責管理和維護HDFS的名稱空間、並且控制檔案的任何讀寫動作<br>
DataNode :負責執行資料讀寫動作，以及執行NameNode的副本策略<br>
ResourceManager(JobTracker) :負責安排MapReduce運算層任務<br>
NodeManager(TaskTracker) :負責執行運算層任務<br>
#### 分配
```
NameNode: h151
SecondNameNode: h152
ResourceManager: h153
Slave: h152, h153, h154
JobHistory: h151
```
# I. create box
1. java1.8
2. key rsa
3. create hadoop user
4. apt-get update & apt-get upgrate

# II. vagrant create machine
## vagrantfile
```bash
Vagrant.configure(2) do |config|
  config.vm.box = "iiiedu/hadoop_node" #load box
  
  config.vm.define "bdse211" do |bdse| # vagrant ssh name
    bdse.vm.hostname = "bdse211" 
	bdse.vm.network :public_network, ip: "192.168.33.211" #ip
    bdse.vm.provider "virtualbox" do |v| 
      v.name = "bdse211" # virtualbox name
      v.cpus = 1 #core
      v.memory = 4096 #ram
    end
	#bdse.vm.provision "shell", path: "scripts/setup.sh" #auto
	bdse.vm.provision "shell", path: "scripts/sethosts.sh", run: "always" #copy hosts to /etc/hosts
  end 
  
  config.vm.define "bdse212" do |bdse| 
    bdse.vm.hostname = "bdse212" 
	bdse.vm.network :public_network, ip: "192.168.33.212"
    bdse.vm.provider "virtualbox" do |v| 
      v.name = "bdse212"  
      v.cpus = 1
      v.memory = 4096 
    end
	#bdse.vm.provision "shell", path: "scripts/setup.sh"
	bdse.vm.provision "shell", path: "scripts/sethosts.sh", run: "always"
  end 

  config.vm.define "bdse213" do |bdse| 
    bdse.vm.hostname = "bdse213" 
	bdse.vm.network :public_network, ip: "192.168.33.213"
    bdse.vm.provider "virtualbox" do |v| 
      v.name = "bdse213" 
      v.cpus = 1
      v.memory = 4096 
    end
	#bdse.vm.provision "shell", path: "scripts/setup.sh"
	bdse.vm.provision "shell", path: "scripts/sethosts.sh", run: "always"
  end
  
end
```
## scripts
### 1.hosts
```
127.0.0.1 localhost
# IP            FQDN                alias
192.168.33.211 bdse211.example.org bdse211
192.168.33.212 bdse212.example.org bdse212
192.168.33.213 bdse213.example.org bdse213
```
###2.sethosts.sh
```bash
#!/bin/bash
cp -f /vagrant/scripts/hosts /etc/hosts
```
# III. In machine
## 1. Install Hadoop
```
wget http://apache.stu.edu.tw/hadoop/common/hadoop-2.7.3/hadoop-2.7.3.tar.gz
tar -xvf hadoop-2.7.2.tar.gz -C /usr/local
mv /usr/local/hadoop-2.7.2 /usr/local/hadoop
chown -R hadoop:hadoop /usr/local/hadoop
```
## 2. Test password-less login 取得第一次登入KEY(不同BOX需新增公鑰，取得免驗證登入)
ssh bdse211@example.org
## 3. Set environment variables 設環境變數
```
nanao ~/.bashrc
    
# Set HADOOP_HOME (hadoop位置)
export HADOOP_HOME=/usr/local/hadoop
# Set JAVA_HOME (java位置)
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
# Add Hadoop bin and sbin directory to PATH (hadoop預設執行程式位置)
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin

source ~/.bashrc (掛載環境變數)
```
## 4. 設定hadoop環境角色(cd /usr/local/hadoop/etc/hadoop/)
### core-site.xml
#### 定義HDFS

預設為/tmp 暫存區 重啟vagrant會不見

NameNode 預設PORT:8020 要玩HA
```xml
<property>
   <name>hadoop.tmp.dir</name>
   <value>/home/hadoop/tmp</value> 
   <description>Temporary Directory.</description>
</property>

<property>
   <name>fs.defaultFS</name>
   <value>hdfs://NameNode.example.org:8020</value>
   <description>Use HDFS as file storage engine</description>
</property>
```
### mapred-site.xml 
#### 定義 
1.hadoop使用yarn做job分配<br>
2.指定jobhistory server存放執行過的job(保留一個月)
cp /usr/local/hadoop/etc/hadoop/mapred-site.xml.template /usr/local/hadoop/etc/hadoop/mapred-site.xml
```xml
<property>
   <name>mapreduce.framework.name</name>
   <value>yarn</value>
</property>
<property>
	 <name>mapreduce.jobhistory.address</name>
	 <value>master3.example.org:10020</value>
</property>
<property>
   <name>mapreduce.jobhistory.webapp.address</name>
	 <value>master3.example.org:19888</value>
</property>
```
### yarn-site.xml 
#### 定義 
1.ResourceManager的機器<br>
2.NodeManager(DataNode)資源使用情況
```xml
<property>
   <name>yarn.nodemanager.aux-services</name>
   <value>mapreduce_shuffle</value>
</property>
<property>
   <name>yarn.nodemanager.resource.memory-mb</name>
   <value>4096</value>
</property>
<property>
   <name>yarn.nodemanager.resource.cpu-vcores</name>
   <value>1</value>
</property>
<property>
	 <name>yarn.resourcemanager.hostname</name>
	 <value>bdse212.example.org</value>
</property>	
```
### slaves
#### 設定Datanode(NodeManager)
```
bdse212.example.org
bdse213.example.org
```
## 4. start Hadoop
#### NameNode
```
hdfs namenode -format (only once)

start-dfs.sh (啟動HDFS)
jps (查看本機執行狀況)
http://master1.example.org:50070 (check NameNode and DataNode)
```
#### ResourceManager(啟動NodeManager)
```
start-yarn.sh
jps
```
#### Historyserver(啟動jobhistory)	
```
mr-jobhistory-daemon.sh start historyserver
jps

http://master1.example.org:8088/cluster (check yarn cluster)
```
# IV. TESTING
```
hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.3.jar pi 30 100

http://master1.example.org:8088/cluster (check running appliction)
```
