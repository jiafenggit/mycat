# mycat
mycat读写分离
只需要读写分离的功能，分库分表的都不需要。

涉及到的配置文件：
1.conf/server.xml
主要配置的是mycat的用户名和密码，mycat的用户名和密码和mysql的用户名密码是分开的，应用连接mycat就用这个用户名和密码。

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://org.opencloudb/">
  <system>
    <!-- 
    <property name="processors">32</property> 
    <property name="processorExecutor">32</property> 
    <property name="serverPort">8066</property> 
    <property name="managerPort">9066</property> 
    -->
  </system>
  <user name="root">
    <property name="password">root</property>
    <property name="schemas">数据库名称</property>
  </user>
</mycat:server>

2.conf/schema.xml
主要配置主从库的数据库连接地址信息，schema里面不能配置table的定义，如果配置了就会检查sql的语法，目前mycat还有很多问题。

 
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://org.opencloudb/">
 
  <schema name="数据库名称" checkSQLschema="false" dataNode="dn1">
  </schema>
 
  <dataNode name="dn1" dataHost="localhost1" database="数据库名称" />
  <dataHost name="localhost1" maxCon="1000" minCon="100" balance="1" 	 dbType="mysql" dbDriver="native">
    <heartbeat>select user()</heartbeat>
    <!-- can have multi write hosts -->
    <writeHost host="10.1.3.50" url="10.1.3.50:3306" user="数据库用户名"  password="数据库密码">
      <!-- can have multi read hosts -->			
      <readHost host="10.1.3.5" url="10.1.3.5:3306" user="数据库用户名" password="数据库密码" />
      <readHost host="10.1.3.6" url="10.1.3.6:3306" user="数据库用户名" password="数据库密码" />
    </writeHost>		
    <!--writeHost host="10.1.3.34" url="10.1.3.34:3306" user="数据库用户名"  password="数据库密码"-->
      <!-- can have multi read hosts -->			
      <!--readHost host="10.1.3.7" url="10.1.3.7:3306" user="数据库用户名" password="数据库密码" /-->
      <!--readHost host="10.1.3.8" url="10.1.3.8:3306" user="数据库用户名" password="数据库密码" /-->
    <!--/writeHost-->
  </dataHost>
</mycat:schema>

高可用性以及读写分离
MyCAT的读写分离机制如下：
• 事务内的SQL，全部走写节点，除非某个select语句以注释/*balance*/开头
• 自动提交的select语句会走读节点，并在所有可用读节点中间随机负载均衡
• 当某个主节点宕机，则其全部读节点都不再被使用，因为此时，同步失败，数据已经不是最新的，MYCAT会采用另外一个主节点所对应的全部读节点来实现select负载均衡。
• 当所有主节点都失败，则为了系统高可用性，自动提交的所有select语句仍将提交到全部存活的读节点上执行，此时系统的很多页面还是能出来数据，只是用户修改或提交会失败。

dataHost的balance属性设置为：
• 0，不开启读写分离机制
• 1，全部的readHost与stand by writeHost参与select语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且M1与 M2互为主备)，正常情况下，M2,S1,S2都参与select语句的负载均衡。
• 2，所有的readHost与writeHost都参与select语句的负载均衡，也就是说，当系统的写操作压力不大的情况下，所有主机都可以承担负载均衡。
一个dataHost元素，表明进行了数据同步的一组数据库，DBA需要保证这一组数据库服务器是进行了数据同步复制的。writeHost相当于Master DB Server，而旗下的readHost则是与从数据库同步的Slave DB Server。当dataHost配置了多个writeHost的时候，任何一个writeHost宕机，Mycat 都会自动检测出来，并尝试切换到下一个可用的writeHost。

MyCAT支持高可用性的企业级特性，根据您的应用特性，可以配置如下几种策略：
• 后端数据库配置为一主多从，并开启读写分离机制。
• 后端数据库配置为双主双从（多从），并开启读写分离机制
• 后端数据库配置为多主多从，并开启读写分离机制
后面两种配置，具有更高的系统可用性，当其中一个写节点（主节点）失败后，Mycat会侦测出来（心跳机制）并自动切换到下一个写节点，MyCAT在任何时候，只会往一个写节点写数据。 
