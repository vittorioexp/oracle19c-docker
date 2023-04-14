# OracleDB Docker Container

A lightweight and configurable Oracle 19c docker image.

Oracle has introduced the concept of container databases (CDB) and pluggable databases (PDB). Containers are used for multi-tenancy and contain pluggable databases. Pluggable databases are what you are probably used to, a self contained database that you connect to. By default, this image creates one CDB, and one PDB within that CDB.

Note that Oracle is shifting away from an SID and using service names instead. PDB use service names, CDB use SIDs. However, usernames for CDB must start with C##, ie C##STEVE. So to make things simple, we are focusing on just using the single PDB.

Oracle have since improved their docker images and I have worked with Oracle on making the the memory configurable (see https://github.com/oracle/docker-images/issues/1575), so all we need now are simplified instructions.

Alternatives
------------

Before proceeding, you may want to consider the following resources:
- `https://hub.docker.com/r/gvenzl/oracle-xe` (a 21c DB)
- `https://hub.docker.com/r/gvenzl/oracle-free` (a 23c DB)
- `https://github.com/oracle/docker-images/tree/main/OracleDatabase`


Before you begin
----------------

Clone `https://github.com/oracle/docker-images`.

Download the Oracle Database 19c binary `LINUX.X64_193000_db_home.zip` from http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html

Put the zip (and DO NOT unzip it) in the `OracleDatabase/SingleInstance/dockerfiles/19.3.0` directory.

In Docker Desktop, ensure you have a large enough amount of memory allocated. These instructions will set the total memory to 4000MB, so make sure Docker has a value higher than that.

Building the docker container
-----------------------------

Open a Unix based terminal (Git Bash, WSL Ubuntu, ...) and run the following:

````
cd OracleDatabase/SingleInstance/dockerfiles
./buildContainerImage.sh -v 19.3.0 -e
````
This will take about 10-15 minutes.

If the build fails saying you are out of space, check how much space you have available on your disk. If it looks ok, prune old Docker images via: 
`yes | docker image prune > /dev/null`


Running the docker container
-----------------------------

To use the sensible defaults:

```
docker run \
--name oracle19c \
-p 1521:1521 \
-p 5500:5500 \
-e ORACLE_PDB=orcl \
-e ORACLE_PWD=password \
-e INIT_SGA_SIZE=3000 \
-e INIT_PGA_SIZE=1000 \
-v /opt/oracle/oradata \
-d \
oracle/database:19.3.0-ee
```

On first run, the database will be created and setup for you.  
Open Docker Dashboard and watch the progress. Then you can connect.

Optionally, you can use the following run commmand to avoid getting "No disk space" issues as you gradually insert more and more data into your database.

```
docker run \
--name oracle19c \
-p 1521:1521 -p 5500:5500 \
-e ORACLE_PDB=orcl \
-e ORACLE_PWD=password \
-e INIT_SGA_SIZE=3000 \
-e INIT_PGA_SIZE=1000 \
-v /Users/<your-username>/path/to/store/db/files/:/opt/oracle/oradata \
-d \
oracle/database:19.3.0-ee
```

All available options follows:

```
   --name:        The name of the container (default: auto generated)
   -p:            The port mapping of the host port to the container port.
                  Two ports are exposed: 1521 (Oracle Listener), 5500 (OEM Express)
   -e ORACLE_SID: The Oracle Database SID that should be used (default: ORCLCDB)
   -e ORACLE_PDB: The Oracle Database Service Name that should be used (default: ORCLPDB1)
   -e ORACLE_PWD: The Oracle Database SYS password (default: auto generated)
   -e INIT_SGA_SIZE: The amount of SGA to allocate to Oracle. This should be 75% of the total memory you want Oracle to use. 
   -e INIT_PGA_SIZE: The amount of PGA to alloxate to oracle. This should be the remaining 25% of the total memory you want Oracle to use.  
   -e ORACLE_CHARACTERSET: The character set to use when creating the database (default: AL32UTF8)
   -v /opt/oracle/oradata
                  The data volume to use for the database.
                  Has to be writable by the Unix "oracle" (uid: 54321) user inside the container!
                  If omitted the database will not be persisted over container recreation.
   -v /Users/<your-username>/path/to/store/db/files/:/opt/oracle/oradata
                  Mount the data volume into one of your local folders.
                  If omitted you might run into a "No disk space" issue at some point as your database keeps growing and docker does not resize its volume.
   -d:            Run in detached mode. You want this otherwise `Ctrl-C` will kill the container.
```

Note that if you do not specify INIT_SGA_SIZE and INIT_PGA_SIZE then Oracle will determine the memory to allocate based on the number of CPUs, available memory in your machine etc, and for desktop environments this could be from about 2000MB to 5000MB. If you want control over this, set the values.


Connecting to the container as root
-----------------------------------


Open a terminal and find your oracle docker container ID, using:
```
docker ps
```

Supposing your ID is 22d7f94dfa3c, run:
```
docker exec -u 0 -it 22d7f94dfa3c /bin/bash
```

You are now connected to the container with privileges.

Resolving compatibilities of the PASSWORD_VERSIONS parameter
------------------------------------------------------------

We need to make the PASSWORD_VERSIONS parameter compatible with all versions.

First thing first, you need to install vi:
```
yum install vi
```

Run the following to edit sqlnet.ora:
```
vi $ORACLE_HOME/network/admin/sqlnet.ora
```

add the following line to the opened file (press "i" to enter the editation mode):
```
SQLNET.ALLOWED_LOGON_VERSION_SERVER=10
```
save and exit (press "esc", and then write ":wq")


Run the following command
```
cat $ORACLE_HOME/network/admin/tnsnames.ora
```
and make sure the output is the following. If not, edit the file.
```
ORCLCDB=localhost:1521/ORCLCDB
ORCL= 
(DESCRIPTION = 
  (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = ORCL)
  )
)
```
Run the following command
```
cat /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
```
and make sure the output is the following. If not, edit the file.
```
LISTENER = 
(DESCRIPTION_LIST = 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1)) 
    (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521)) 
    (CONNECT_DATA = (SERVICE_NAME = orcl)) 
  ) 
)

DEDICATED_THROUGH_BROKER_LISTENER=ON
DIAG_ADR_ENABLED = off
```


Connecting to the database as sys
---------------------------------

To connect as admin (sysdba) to the oracle database, run:
```
sqlplus sys/password@orcl as sysdba
```

Creating a new user with privileges
-----------------------------------

If the connection is successful, run the following to create a user and grant privileges:
```
CREATE USER SIB2000 IDENTIFIED BY password;
GRANT ALL PRIVILEGES TO SIB2000;
```


Now open a client (such as dbeaver) and try to connect to the oracle database using the followings:
```
Host: localhost
Port: 1521
Database: orcl (Service Name)
Authentication: Oracle Database Native
Username: SIB2000
Role: Normal
Password: password
```



