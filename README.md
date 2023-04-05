# oracle19c-docker
A lightweight and configurable Oracle 19c docker image.

Oracle has introduced the concept of container databases (CDB) and pluggable databases (PDB). Containers are used for multi-tenancy and contain pluggable databases. Pluggable databases are what you are probably used to, a self contained database that you connect to. By default, this image creates one CDB, and one PDB within that CDB.

Note that Oracle is shifting away from an SID and using service names instead. PDB use service names, CDB use SIDs. However, usernames for CDB must start with C##, ie C##STEVE. So to make things simple, we are focusing on just using the single PDB.

I used to maintain a standalone repository for building this image. It was originally based on the official images, like my Oracle 12c one here: https://github.com/steveswinsburg/oracle12c-docker

Oracle have since improved their docker images and I have worked with Oracle on making the the memory configurable (see https://github.com/oracle/docker-images/issues/1575), so all we need now are simplified instructions.

Before you begin
----------------

1. Clone `https://github.com/oracle/docker-images`.
1. Download the Oracle Database 19c binary `LINUX.X64_193000_db_home.zip` from http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html
1. Put the zip in the `OracleDatabase/SingleInstance/dockerfiles/19.3.0` directory. **Do not unzip it.**
1. In Docker Desktop, ensure you have a large enough amount of memory allocated. These instructions will set the total memory to 4000MB, so make sure Docker has a value higher than that.

Building
--------


````
cd OracleDatabase/SingleInstance/dockerfiles
./buildContainerImage.sh -v 19.3.0 -e
````

If the build fails saying you are out of space, check how much space you have available on your disk. If it looks ok, prune old Docker images via: 
`yes | docker image prune > /dev/null`

Running
-------

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

On first run, the database will be created and setup for you. This will take about 10-15 minutes. Open Docker Dashboard and watch the progress. Then you can connect.

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

Configuration
-------------

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

Connecting to Oracle
--------------------

Once the container has been started you can connect to it like any other database. Note we are using `Service Name` and not the SID (since PDB uses Service Name).

For example, SQL Developer:
```
Hostname: localhost
Port: 1521
Service Name: <your service name>
Username: sys
Password: <your password>
Role: AS SYSDBA

```

Changing the password
---------------------

Note: If you did not set the `ORACLE_PWD` parameter, check the docker run output for the password.

The password for the SYS account can be changed via the `docker exec` command. Note, the container has to be running:

First run `docker ps` to get the container ID. Then run:
`docker exec <container id> ./setPassword.sh <new password>`



Connecting to the container as root
-----------------------------------


Open a terminal.

Find your oracle docker container ID, using:
```
docker ps
```

Supposing your ID is 22d7f94dfa3c, run:
```
docker exec -u 0 -it 22d7f94dfa3c /bin/bash
```

You are now connected to the container with privileges.


Now you need to install vi:
```
yum install vi
```
We need to make the PASSWORD_VERSIONS parameter compatible with all versions, so run the following to edit sqlnet.ora:
```
vi $ORACLE_HOME/network/admin/sqlnet.ora
```

add the following line (press "i" to enter the editation mode):
```
SQLNET.ALLOWED_LOGON_VERSION_SERVER=10
```
save and exit (press "esc", and then write ":wq")



To connect as admin (sysdba) to the oracle database, run:
```
sqlplus sys/password@orcl as sysdba
```

If successful, run the following to create a user and grant privileges:
```
CREATE USER SIB2000 IDENTIFIED BY password;
GRANT ALL PRIVILEGES TO SIB2000;
```


Now open dbeaver and try to connect to the oracle database using the followings:
```
Host: localhost
Port: 1521
Database: orcl (Service Name)
Authentication: Oracle Database Native
Username: SIB2000
Role: Normal
Password: password
```



