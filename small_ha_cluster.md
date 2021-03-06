{{#template name='small_ha_cluster'}}
# Small High Availability Cluster Setup Guide

This is a guide to setting up PredictionIO in a 3 node cluster with all services running on the 3 cluster machines.

In this guide all services are setup with multiple or standby masters in true clustered mode. To make  High Availability complete, a secondary master would need to be setup for HDFS (not described here). Elasticsearch and HBase are setup in High Availability mode (HA) using this guide.

#### Other Guides:

 Choose the guide that best fits your needs.

{{> piosinglemachineguide}}
{{> piodistributedguide}}

## Requirements

In this guide, all servers share all services, except PredictionIO, which runs only on the master server. Setup of multiple EventServers and PredictionServers is done with load-balancers and is out of the scope of this guide.

Here we'll install and setup:

- Hadoop {{> hdfsversion}} (Clustered, standby master needed for full HA)
- Spark {{> sparkversion}} (Clustered)
- Elasticsearch {{> elasticsearchversion}} (Clustered, standby master)
- HBase {{> hbaseversion}} (Clustered, standby master) due to a bug in 1.1.2 and earlier HBase it is advised you move to {{> pioversion}} installation [instructions here](/docs/pio_quickstart).
- Universal Recommender [here](/docs/ur_quickstart)
- 'Nix server, some instructions below are specific to Ubuntu, a Debian derivative and Mac OS X. Using Windows it is advised that you run a VM with a Linux OS.


## 1. Setup User, SSH, and host naming on All Hosts:

1.1 Create user for PredictionIO `aml` in each server

    adduser aml # Give it some password

1.2 Give the `aml` user sudoers permissions and login to the new user. This setup assumes the aml user as the **owner of all services** including Spark and Hadoop (HDFS).

**Note**: to do the following you may need to create a group called `sudo` and edit `/etc/sudoers` to add `%sudo  ALL = (ALL) NOPASSWD: ALL`

    usermod -a -G sudo aml
    sudo su aml # or exit and login as the aml user

Notice that we are now logged in as the `aml` user

1.3 Setup passwordless ssh between all hosts of the cluster. This is a combination of adding all public keys to `authorized_keys` and making sure that `known_hosts` includes all cluster hosts, including any host to itself. There must be no prompt generated when any host tries to connect via ssh to any other host. **Note:** The importance of this cannot be overstated! If ssh does not connect without requiring a password and without asking for confirmation **nothing else in the guide will work!**

1.4 Modify `/etc/hosts` file and name each server. Don't use "localhost" or "127.0.0.1" but use either the lan DNS name or static IP address.

    # Use IPs for your hosts.
    10.0.0.1 some-master
    10.0.0.2 some-slave-1
    10.0.0.3 some-slave-2

## 2. Download Services on all Hosts:

Download everything to a temp folder like `/tmp/downloads`, we will later move them to the final destinations. **Note**: You may need to install `wget` with `yum` or `apt-get`.

2.1 Download {{> hdfsdownload}}

2.2 Download {{> sparkdownload}}

2.3 Download {{> esdownload}}

2.4 Download {{> hbasedownload}}

2.5 Clone the ActionML version of PredictionIO from its root repo into `~/pio-aml`

    git clone https://github.com/actionml/PredictionIO.git pio-aml
    cd ~/pio-aml
    git checkout master #get the latest branch

2.6 Clone Universal Recommender Template from its root repo into `~/universal` or do similar for any other template.

    git clone https://github.com/actionml/template-scala-parallel-universal-recommendation.git universal
	cd ~/universal
	git checkout master # or get the tag you want


## 3. Setup Java 1.7 or 1.8

3.1 Install Java OpenJDK or Oracle JDK for Java 7 or 8, the JRE version is not sufficient.

    sudo apt-get install openjdk-8-jdk
    # for centos
    # sudo yum install java-1.8.0-openjdk
    
**Note**: on Centos/RHEL you may need to install java-1.8.0-openjdk-devel if you get complaints about missing `javac` or `javadoc`

3.2 Check which versions of Java are installed and pick a 1.7 or greater.

    sudo update-alternatives --config java

3.3 Set JAVA_HOME env var.

Don't include the `/bin` folder in the path. This can be problematic so if you get complaints about JAVA\_HOME you may need to change xxx-env.sh depending on which service complains. For instance `hbase-env.sh` has a JAVA\_HOME setting if HBase complains when starting.

    vim /etc/environment
    # add the following
    export JAVA_HOME=/path/to/open/jdk/jre
    # some would rather add JAVA_HOME to /home/aml/.bashrc

##4. Create Folders:

4.1 Create folders in `/opt`

    mkdir /opt/hadoop
	mkdir /opt/spark
	mkdir /opt/elasticsearch
	mkdir /opt/hbase
	chown aml:aml /opt/hadoop
	chown aml:aml /opt/spark
	chown aml:aml /opt/elasticsearch
	chown aml:aml /opt/hbase

##5. Extract Services

5.1 Inside the `/tmp/downloads` folder, extract all downloaded services.

{{> setsymlinks}}


##6. Setup Clustered services

### 6.1. Setup Hadoop Cluster

Read [this tutorial](http://www.tutorialspoint.com/hadoop/hadoop_multi_node_cluster.htm)

- Files config: this  defines the defines where the root of HDFS will be. To write to HDFS you can reference this location, for instance in place of a local path like `file:///home/aml/file` you could read or write `hdfs://some-master:9000/user/aml/file`

  - `etc/hadoop/core-site.xml`

```
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://some-master:9000</value>
  </property>
</configuration>
```

- `etc/hadoop/hadoop/hdfs-site.xml` This sets the actual filesystem location that hadoop will use to save data and how many copies of the data to be kept. In case of storage corruption, hadoop will restore from a replica and eventually restore replicas. If a server goes down, all data on that server will be re-created if you have at a `dfs.replication` of least 2.

```
<configuration>
   <property>
      <name>dfs.data.dir</name>
      <value>file:///usr/local/hadoop/dfs/name/data</value>
      <final>true</final>
   </property>

   <property>
      <name>dfs.name.dir</name>
      <value>file:///usr/local/hadoop/dfs/name</value>
      <final>true</final>
   </property>

   <property>
      <name>dfs.replication</name>
      <value>2</value>
   </property>
</configuration>
```

  - `etc/hadoop/masters` One master for this config.

	```
	some-master
	```

  - `etc/hadoop/slaves` Slaves for HDFS means they have datanodes so the master may also host data with this config

        some-master
        some-slave-1
        some-slave-2


  - `etc/hadoop/hadoop-env.sh` make sure the following values are set

    ```
    export JAVA_HOME=${JAVA_HOME}
    # this has been set for hadoop historically but not sure it is needed anymore
    export HADOOP_OPTS=-Djava.net.preferIPv4Stack=true
    export HADOOP_CONF_DIR=${HADOOP_CONF_DIR:-"/etc/hadoop"}
    ```

 - Format Namenode

      bin/hadoop namenode -format

    This will result actions logged to the terminal, make sure there are no errors

 - Start dfs servers only.

        sbin/start-dfs.sh

    Do not use `sbin/start-all.sh` because it will needlessly start mapreduce and yarn. These can work together with PredictionIO but for the purposes of this guide they are not needed.

 - Create required HDFS directories

        hdfs dfs -mkdir /hbase
        hdfs dfs -mkdir /zookeeper
        hdfs dfs -mkdir /models
        hdfs dfs -mkdir /user
        hdfs dfs -mkdir /user/aml # will be like ~ for user "aml"


#### 6.2. Setup Spark Cluster.
- Read and follow [this tutorial](http://spark.apache.org/docs/latest/spark-standalone.html) The primary thing that must be setup is the masters and slaves, which for our purposes will be the same as for hadoop
-  `conf/masters` One master for this config.

	```
	some-master
	```

  - `conf/slaves` Slaves for Spark means they are workers so the master be included

        some-master
        some-slave-1
        some-slave-2


- Start all nodes in the cluster

    `sbin/start-all.sh`


#### 6.3. Setup Elasticsearch Cluster

- Change the `/usr/local/elasticsearch/config/elasticsearch.yml` file as shown below. This is minimal and allows all hosts to act as backup masters in case the acting master goes down. Also all hosts are data/index nodes so can respond to queries and host shards of the index.

    The `cluster.name` defines the Elasticsearch machines that form a cluster. This is used by Elasticsearch to discover other machines and should not be set to any other PredictionIO id like appName. It is also important that the cluster name not be left as default or your machines may join up with others on the same LAN.

```
cluster.name: some-cluster-name
discovery.zen.ping.multicast.enabled: false # most cloud services don't allow multicast
discovery.zen.ping.unicast.hosts: ["some-master", "some-slave-1", "some-slave-2"] # add all hosts, masters and/or data nodes
```

- copy Elasticsearch and config to all hosts using `scp -r /opt/elasticsearch/... aml@some-host://opt/elasticsearch`. Like HBase, all hosts are identical.

#### 6.4. Setup HBase Cluster (abandon hope all ye who enter here)

This [tutorial](https://hbase.apache.org/book.html#quickstart_fully_distributed) is the **best guide**, many others produce incorrect results . The primary thing to remember is to install and configure on a single machine, adding all desired hostnames to `backupmasters`, `regionservers`, and to the `hbase.zookeeper.quorum` config param, then copy **all code and config** to all other machines with something like `scp -r ...` Every machine will then be identical.

6.4.1 Configure with these changes to `/usr/local/hbase/conf`

  - `conf/hbase-site.xml`

```
<configuration>
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://some-master:9000/hbase</value>
    </property>

    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>

    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>hdfs://some-master:9000/zookeeper</value>
    </property>

    <property>
        <name>hbase.zookeeper.quorum</name>
        <value>some-master,some-slave-1,some-slave-2</value>
    </property>

    <property>
        <name>hbase.zookeeper.property.clientPort</name>
        <value>2181</value>
    </property>
</configuration>
```

  - `conf/regionservers`

		some-master
		some-slave-1
		some-slave-2

  - `conf/backupmasters`

        some-slave-1

  - `conf/hbase-env.sh`

		export JAVA_HOME=${JAVA_HOME}
		export HBASE_MANAGES_ZK=true # when you want HBase to manage zookeeper

6.4.2 Start HBase

    bin/start-hbase.sh

At this point you should see several different processes start on the master and slaves including regionservers and zookeeper servers. If there is an error check the log files referenced in the error message. These log files may reside on a different host as indicated in the file's name.

**Note:** It is strongly recommend to setup these files in the master `/usr/local/hbase` folder and then copy **all** code and sub-folders or the to the slaves. All members of the cluster must have the same code and config


##7. Setup PredictionIO

Setup PIO on the master or on all servers (if you plan to use a load balancer). The Setup **must not** use the install.sh since you are using clustered services and that script only supports a standalone machine.

7.1 Build PredictionIO

We put PredictionIO in `/home/aml/pio` Change to that location and run

    ./make-distribution

This will create an artifact for PredictionIO

7.2 Setup Path for PIO commands

Add PIO to the path by editing your `~/.bashrc` on the master. Here is an example of the important values I have in the file. After changing it remember for execute `source ~/.bashrc` to get the changes into the running shell.

**Note:** Some of the service setup may ask for you to add other things so the ones below are only for PIO itself and the Universal Recommender.

    # Java
	export JAVA_OPTS="-Xmx4g"
    # You may need to experiment with this setting if you get 
    # "out of memory error"" for the driver, executor memory and 
    # Spark settings can be set in the
	# sparkConf section of engine.json

	# Spark
	# this tells PIO which host to use for Spark
	export MASTER=spark://some-master:7077
	export SPARK_HOME=/usr/local/spark

	# pio-aml
	export PATH=$PATH:/usr/local/pio-aml/bin:/usr/local/pio-aml

Run `source ~/.bashrc` to get changes applied.

7.3 Setup PredictionIO to connect to the services

You have PredictionIO in `~/aml` so edit ~/pio-aml/conf/pio-env.sh to have these settings:

    # Safe config that will work if you expand your cluster later
    SPARK_HOME=/usr/local/spark
    ES_CONF_DIR=/usr/local/elasticsearch
    HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
    HBASE_CONF_DIR=/usr/local/hbase/conf
    
    
    # Filesystem paths where PredictionIO uses as block storage.
    PIO_FS_BASEDIR=$HOME/.pio_store
    PIO_FS_ENGINESDIR=$PIO_FS_BASEDIR/engines
    PIO_FS_TMPDIR=$PIO_FS_BASEDIR/tmp
    
    # PredictionIO Storage Configuration
    #
    # This section controls programs that make use of PredictionIO's built-in
    # storage facilities. Default values are shown below.
    
    # Storage Repositories
    
    # Default is to use PostgreSQL but for clustered scalable setup we'll use
    # Elasticsearch
    PIO_STORAGE_REPOSITORIES_METADATA_NAME=pio_meta
    PIO_STORAGE_REPOSITORIES_METADATA_SOURCE=ELASTICSEARCH
    
    PIO_STORAGE_REPOSITORIES_EVENTDATA_NAME=pio_event
    PIO_STORAGE_REPOSITORIES_EVENTDATA_SOURCE=HBASE
    
    # Need to use HDFS here instead of LOCALFS to enable deploying to 
    # machines without the local model
    PIO_STORAGE_REPOSITORIES_MODELDATA_NAME=pio_model
    PIO_STORAGE_REPOSITORIES_MODELDATA_SOURCE=HDFS
    
    # Storage Data Sources, lower level that repos above, just a simple storage API
    # to use
    
    # Elasticsearch Example
    PIO_STORAGE_SOURCES_ELASTICSEARCH_TYPE=elasticsearch
    PIO_STORAGE_SOURCES_ELASTICSEARCH_HOME=/usr/local/elasticsearch
    # The next line should match the ES cluster.name in ES config
    PIO_STORAGE_SOURCES_ELASTICSEARCH_CLUSTERNAME=some-cluster-name
    
    # For clustered Elasticsearch (use one host/port if not clustered)
    PIO_STORAGE_SOURCES_ELASTICSEARCH_HOSTS=some-master,some-slave-1,some-slave-2
    PIO_STORAGE_SOURCES_ELASTICSEARCH_PORTS=9300,9300,9300
    
    PIO_STORAGE_SOURCES_HDFS_TYPE=hdfs
    PIO_STORAGE_SOURCES_HDFS_PATH=hdfs://some-master:9000/models
    
    # HBase Source config
    PIO_STORAGE_SOURCES_HBASE_TYPE=hbase
    PIO_STORAGE_SOURCES_HBASE_HOME=/usr/local/hbase
    
    # Hbase clustered config (use one host/port if not clustered)
    PIO_STORAGE_SOURCES_HBASE_HOSTS=some-master,some-slave-1,some-slave-2
    PIO_STORAGE_SOURCES_HBASE_PORTS=0,0,0

Then you should be able to run

    pio-start-all
    pio status

The status of all the stores is checked and will be printed but no check is made of the HDFS or Spark services so check them separately by looking at their GUI status pages. They are here:

 - HDFS: http://some-master:50070
 - Spark: http://some-master:8080

##8. Setup Your Template

See the template setup instructions. The Universal Recommender can be installed with its [quickstart](/docs/ur_quickstart).

## 9. Scaling for Load Balancers

See [**PredictionIO Load Balancing**](pio_load_balancing)

{{/template}}