<h1 align="center">mongo-connector使用备忘</h1>
<h5 align="right"> 2017年10月31号</h5>
    
### 1、 mongo-connector的config文件主要参数介绍
<pre><code>
   * "mainAddress": "localhost:27017"   #mongodb的连接url
       
   * "oplogFile": "/var/log/mongo-connector/oplog.timestamp"  # mongo-connector的timestamp文件存储位置，修改为自己定义的文件目录
       
   * "noDump": false  #default false 是否从开始同步oplog
      
   * "batchSize": -1    #在更新timestamp文件前处理的文件数量，-1默认，可以修改为1000
      
   * "verbosity": 0   #debug level  0 ERROR  1 Warning 2 info 3 debug
      
   * "continueOnError": false   #遇到错误是否继续，建议修改为true

   * "logging": {
        "type": "file",  #log类型 默认文件类型
        "filename": "/var/log/mongo-connector/mongo-connector.log",  #log文件存储位置和名字，可以修改为自己的日志文件存储目录
    },

    * "authentication": {
        "__adminUsername": "username",  #连接mongodb的账户,__表示不启用该参数，修改为为mongodb的账户
        "__password": "password",   #mongodb账户密码，修改为mongodb的密码
    },

    * "__fields": ["field1", "field2", "field3"],  #需要同步的mongo中的字段，“_id”必须放在第一个字段，且必须有该字段
    * 
    * "namespaces": {  #mongo库表到es的映射
        "include": ["mydb.mytb"],  #mongodb中的库表名字
        "mapping": {
            "mydb.mytb":"myindex.mytype"  #mongodb映射到es的索引名字和类型名字
        }
    }
    
    * "docManagers": [
        {
            "docManager": "elastic_doc_manager",  #同步到的目标数据源类型es,mongodb,solr,自定义，默认是es，目前使用的版本为elastic2_doc_manager，mongo用mongo_doc_manager，solr用solr_doc_manager
            "targetURL": "localhost:9200", #目标数据源连接url
            "__bulkSize": 1000,  #每次同步数据量
            "__uniqueKey": "_id",  #主键
            "__autoCommitInterval": null #是否自动提交
        }
    ]
}
</code></pre>
  
### 2、安装mongo-connector

以linux为例：

<pre><code>
cd /usr/local/
git clone https://github.com/mongodb-labs/mongo-connector.git   #需要提前安装git
</code></pre>

如果没有安装git，在github上下载zip包，解压缩，然后安装。
下载地址：
<pre><code>
https://github.com/mongodb-labs/mongo-connector/releases
</code></pre>

安装：
<pre><code>
cd mongo-connector-2.5.1
python setup.py install
</code></pre>

安装成功会在/usr/local/lib/python2.7/site-packages中添加一些文件，如：
<pre><code>
mongo\_connector-2.5.1-py2.7.egg    
mongo_connector
</code></pre>

### 3、安装elastic2\_doc_manager

<pre><code>
cd /usr/local/mongo-connector/
sudo git clone https://github.com/mongodb-labs/elastic2-doc-manager
</code></pre>

或者在github上下载zip包，解压缩，
下载地址：

<pre><code>
https://github.com/mongodb-labs/elastic2-doc-manager/releases
</code></pre>
安装：
<pre><code>
cd /usr/local/mongo-connector-2.5.1/elastic2-doc-manager-0.3.0
python setup.py install
</code></pre>

安装成功后会在python的lib安装目录下出现elastic2-doc-manager的一些文件，如：

<pre><code>
elastic2_doc_manager-0.3.0-py2.7.egg
elasticsearch
elasticsearch-2.3.0.dist-info
</code></pre>

如果要对从mongodb中的数据在转入es的过程中进行格式转换，可以修改
<pre><code>
/usr/local/lib/python2.7/site-packages/elastic2\_doc_manager-0.3.0-py2.7.egg/mongo\_connector/doc\_managers/elastic2\_doc_manager.py
</code></pre>
文件。注意：该目录会由于python的安装目录不同而不同。

### 4、把mongo-connector安装为服务

<pre><code>
cd /usr/local/mongo-connector-2.5.1
python setup.py install_service
</code></pre>

然后mongo-connector的config文件在/etc/mongo-connector.json，修改里面的配置项，即可使用。

<pre><code>
service mongo-connector start   #启动mongo-connector
service mongo-connector stop   #停止
service mongo-connector status #查看mongo-connector的运行状态
</code></pre>

mongo-connector启动以后，会在mongo-connector的timestamp文件存储位置，也就是oplog所在的目录生成一个oplog.timestamp文件，存储同步oplog的位置时间戳。

如果需要从开始同步oplog，可以删除该文件，然后重启mongo-connector，如果想要从某一个时刻重新同步数据，可以停止mongo-connector服务，修改这个timestamp文件里面的时间戳，修改方法见：
<pre><code>
https://github.com/mongodb-labs/mongo-connector/issues/305
</code></pre>
时间戳需要进行格式转换。

### 5、多个mongodb表入es为多个index，使用多个进程的方法

* 复制/usr/local/lib/python2.7/site-packages/elastic2\_doc_manager-0.3.0-py2.7.egg/mongo\_connector/doc\_managers/elastic2\_doc_manager.py，重命名该文件为test\_doc\_manager.py，修改文件内容解析对应的数据。elastic2\_doc_manager.py文件所在的目录会因为安装方法不同在不同的目录。

* 在/etc/目录下面复制mongo-connector.json文件，重命名为test-connector.json，修改test-connector.json里面的oplogFile存储位置和log文件存储位置，避免不同的进程冲突，修改docManager为test\_doc_manager.py。

* 启动mongo-connector的时候使用python进行启动，如：

<pre><code>
nohup /usr/bin/python -m mongo_connector.connector -c /etc/test-connector.json &
</code></pre>

### 6、mongodb to mongo的配置

如果需要实现mongodb to mongodb的数据同步，只需要修改config文件的docManager即可。如：
<pre><code>
"docManagers": [
        {
            "docManager": "mongo_doc_manager",  
            "targetURL": "localhost:9200", #目标数据源连接url修改为目标mongo的连接
            "bulkSize": 1000,  #每次同步数据量
            "uniqueKey": "\_id",  #主键
            "autoCommitInterval": null #是否自动提交
        }
</code></pre>
