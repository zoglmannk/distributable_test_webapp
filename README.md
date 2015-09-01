# distributable_test_webapp
Are you configuring Wildfly's JGroups subsystem and need something to force JGroups to startup upon deployment? Look no further. The "distributable" element causes JGroups to startup.


If you are looking for a Wildfly JGroups JDBCPING example, look no further. Below are the relevant snipits from the Wildfly configuration. Note that the SQL is MySQL specific. The ```driver``` element will be different depending on how you installed/deployed the MySQL driver. I was lazy and just copied the MySQL driver, ```mysql-connector-java-5.1.36-bin.jar```, to the deployments directory. And obviously the IP address in the connection-url will need changed. Change it to wherever the MySQL DB is running.
```
...
...
                <datasource jndi-name="java:/jgroups" pool-name="jgroups" enabled="true" use-java-context="true" use-ccm="true">
                    <connection-url>jdbc:mysql://172.17.0.1:3306/jgroups</connection-url>
                    <driver>mysql-connector-java-5.1.36-bin.jar_com.mysql.jdbc.Driver_5_1</driver>
                    <driver-class>com.mysql.jdbc.Driver</driver-class>
                    <security>
                        <user-name>jgroups</user-name>
                        <password>some_password_to_your_db</password>
                    </security>
                </datasource>
...
...
...
        <subsystem xmlns="urn:jboss:domain:jgroups:2.0" default-stack="jdbcping">
               <stack name="jdbcping">
                    <transport type="TCP" socket-binding="jgroups-tcp"/>
                    <protocol type="JDBC_PING">
                      <property name="datasource_jndi_name">java:/jgroups</property>
                      <property name="initialize_sql">
CREATE TABLE IF NOT EXISTS jgroups.jgroupsping (own_addr varchar(200) NOT NULL, cluster_name varchar(200) NOT NULL, ping_data varbinary(5000) DEFAULT NULL, PRIMARY KEY (own_addr, cluster_name)) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
                      </property>
                      <property name="insert_single_sql">
INSERT INTO jgroups.jgroupsping (own_addr, bind_addr, created, cluster_name, ping_data) values (?, '${env.JBOSS_BIND_ADDRESS_PUBLIC:127.0.0.1}',CURRENT_TIMESTAMP, ?, ?)
                      </property>
                      <property name="delete_single_sql">
DELETE FROM jgroups.jgroupsping WHERE own_addr=? AND cluster_name=?
                      </property>
                      <property name="select_all_pingdata_sql">
SELECT ping_data FROM jgroups.jgroupsping WHERE cluster_name=?
                      </property>
                    </protocol>
                    <protocol type="MERGE2"/>
                    <protocol type="FD_SOCK" socket-binding="jgroups-tcp-fd"/>
                    <protocol type="FD"/>
                    <protocol type="VERIFY_SUSPECT"/>
                    <protocol type="BARRIER"/>
                    <protocol type="pbcast.NAKACK"/>
                    <protocol type="UNICAST2"/>
                    <protocol type="pbcast.STABLE"/>
                    <protocol type="pbcast.GMS"/>
                    <protocol type="UFC"/>
                    <protocol type="MFC"/>
                    <protocol type="FRAG2"/>
                    <protocol type="RSVP"/>
                </stack>
        </subsystem>
...
...
...
```

For this example to work, MySQL should be configured something like the following
```
create database jgroups;
create user 'jgroups'@'%' identified by 'some_password_to_your_db';
grant all on jgroups.* to 'jgroups'@'%';

use jgroups;

CREATE TABLE jgroups.jgroupsping (
   own_addr varchar(200) NOT NULL,
   bind_addr varchar(200) NOT NULL,
   created timestamp NOT NULL,
   cluster_name varchar(200) NOT NULL,
   ping_data blob,
   constraint PK_JGROUPSPING PRIMARY KEY (own_addr, cluster_name));
```

If you are using Docker and want a quick MySQL instance, you could using something like the following.
```
docker run --name wildfly-assist-mysql -e MYSQL_ROOT_PASSWORD=something -d mysql:5.7.8
```

Then you could connect to it with the following and setup the database.
```
docker run -it --link wildfly-assist-mysql:mysql --rm mysql:5.7.8 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
```
