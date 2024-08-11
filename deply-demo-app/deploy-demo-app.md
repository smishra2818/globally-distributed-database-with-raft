# Deploy the Demo Application

## Introduction

To learn more about Oracle Globally Distributed Database, download and deploy the system-managed shard demo application. The demo application uses the shard environment and schema you have just created to simulate the workload of an online retail store. 

Estimated Lab Time: 30 minutes.

### Objectives

In this lab, you will perform the following steps:
- Setup and Configure the sharding demo application
- Start the workload to load the demo data

### Prerequisites

This lab assumes you have already completed the following:
- Globally Distributed Database Deployment
- Create Demo Sample Schema

## **Task 1:** Setup and Configure the Sharding Demo Application

1. Login to the gsm host.

    ```
    $ ssh -i labkey opc@<gsmhost public ip>
    Activate the web console with: systemctl enable --now cockpit.socket
    
    Last login: Sun Aug 11 00:49:24 2024 from 222.129.1.52
    [opc@gsmhost ~]$ 
    ```
    
    Switch to oracle user.
    
    ```
    [opc@gsmhost ~]$ sudo su - oracle
    Last login: Sun Aug 11 00:49:28 GMT 2024 on pts/3
    [oracle@gsmhost ~]$ 
    ```

   

2. Download the `sdb-demo-app.zip`  file. 

    ```
    [oracle@gsmhost ~]$ https://github.com/minqiaowang/globally-distributed-database-with-raft/raw/main/create-sample-schema/sdb-demo-app.zip
    ```

   

3. Unzip the file. This will create `sdb_demo_app` directory under the `/home/oracle`

    ```
    [oracle@gsmhost ~]$ unzip sdb-demo-app.zip
    ```

   

4. Change to the `sdb_demo_app/sql` directory.

    ```
    [oracle@gsmhost ~]$ cd ./sdb_demo_app/sql
    [oracle@gsmhost sql]$
    ```

   

5. View the content of the `demo_app_ext.sql`. Make sure the connect string is correct.

    ```
    [oracle@gsmhost sql]$ cat demo_app_ext.sql
    -- Create catalog monitor packages
    connect sys/your-own-sys-password@sdbsc0:1521/sdbpdb as sysdba
    @catalog_monitor.sql
    
    connect app_schema/App_Schema_Pass_123@sdbsc0:1521/sdbpdb;
    
    alter session enable shard ddl;
    
    CREATE OR REPLACE VIEW SAMPLE_ORDERS AS
      SELECT OrderId, CustId, OrderDate, SumTotal FROM
        (SELECT * FROM ORDERS ORDER BY OrderId DESC)
          WHERE ROWNUM < 10;
    
    alter session disable shard ddl;
    
    -- Allow a special query for dbaview
    connect sys/your-own-sys-password@sdbsc0:1521/sdbpdb as sysdba
    
    -- For demo app purposes
    grant shard_monitor_role, gsmadmin_role to app_schema;
    
    alter session enable shard ddl;
    
    create user dbmonuser identified by TEZiPP4_MsLLL_1;
    grant connect, alter session, shard_monitor_role, gsmadmin_role to dbmonuser;
    
    grant all privileges on app_schema.products to dbmonuser;
    grant read on app_schema.sample_orders to dbmonuser;
    
    alter session disable shard ddl;
    -- End workaround
    
    exec dbms_global_views.create_any_view('SAMPLE_ORDERS', 'APP_SCHEMA.SAMPLE_ORDERS', 'GLOBAL_SAMPLE_ORDERS', 0, 1);
    [oracle@sdbsd0 sql]$ 
    ```

   

6. Using SQLPLUS to run the script.

    ```
    [oracle@sdbsd0 sql]$ <copy>sqlplus /nolog</copy>
    
    SQL*Plus: Release 19.0.0.0.0 - Production on Sat Jan 23 07:41:39 2021
    Version 19.3.0.0.0
    
    Copyright (c) 1982, 2019, Oracle.  All rights reserved.
    
    SQL> <copy>@demo_app_ext.sql</copy>
    ```
    
7. Make sure there is no error return. 

8. Exit the sqlplus. Change directory to the `sdb_demo_app`.

    ```
    [oracle@sdbsd0 sql]$ <copy>cd ~/sdb_demo_app</copy>
    [oracle@sdbsd0 sdb_demo_app]$
    ```

   

9. View the `demo.properties` file, make sure it's like the following. 

    ```
    name=demo
    connect_string=(ADDRESS_LIST=(LOAD_BALANCE=off)(FAILOVER=on)(ADDRESS=(HOST=localhost)(PORT=1522)(PROTOCOL=tcp)))
    monitor.user=dbmonuser
    monitor.pass=TEZiPP4_MsLLL_l
    app.service.write=oltp_rw_srvc.sdb.oradbcloud
    #app.service.write=rw.orasdb.oradbcloud
    app.service.readonly=oltp_rw_srvc.sdb.oradbcloud
    #app.service.readonly=ro.orasdb.oradbcloud
    app.user=app_schema
    app.pass=App_Schema_Pass_123
    app.threads=7
    ```

   

   

## **Task 2:** Start the workload

1. Start the workload by executing command: `./run.sh demo`.

    ```
    [oracle@sdbsd0 sdb_demo_app]$ <copy>./run.sh demo</copy>
    ```

   

2. The result likes the following.

    ```
    Performing initial fill of the products table...
    Syncing shards...
     RO Queries | RW Queries | RO Failed  | RW Failed  | APS 
              0            0            0            0            0
              0            0            0            0            0
              0            0            0            0            0
            130            0            0            0           40
            130            0            0            0            0
            175           16            0            0           15
           1510          275            0            0          444
           3761          597            0            0          758
           6393         1011            0            0          894
           8829         1373            0            0          818
          11114         1770            0            0          774
          13502         2186            0            0          802
          15981         2618            0            0          839
          18671         3086            0            0          918
          21330         3481            0            0          901
          23936         3927            0            0          888
          26996         4368            0            0         1038
          29798         4795            0            0          952
          32092         5180            0            0          786
    ```

   

3. Wait several minutes, remember the APS (Transcation per Second).

    ```
    RO Queries | RW Queries | RO Failed  | RW Failed  | APS 
         649078       107862            0            0         1332
         652664       108474            0            0         1361
         656194       109085            0            0         1338
         659527       109634            0            0         1262
         662797       110231            0            0         1218
         666327       110824            0            0         1345
         669844       111398            0            0         1328
         673731       111946            0            0         1481
         676797       112513            0            0         1154
         680017       113077            0            0         1221
         683444       113685            0            0         1277
         686942       114306            0            0         1327
         690470       114894            0            0         1332
         693877       115493            0            0         1273
         697395       116064            0            0         1320
         700774       116677            0            0         1278
         703757       117163            0            0         1143
    ```

   

4. Open another terminal, connect to the gsm host,.

    ```
    $ ssh -i labkey opc@<gsmhost public ip>
    Last login: Sat Jan 23 07:27:57 2021 from 59.66.120.23
    -bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
    
    [opc@sdbsd0 ~]$
    ```
    
    Switch to oracle user.
    
    ```
    [opc@sdbsd0 ~]$ <copy>sudo su - oracle</copy>
    Last login: Sat Jan 23 07:29:32 GMT 2021 on pts/0
    
    [oracle@sdbsd0 ~]$
    ```
    
     Change the directory to `sdb_demo_app`.
    
    ```
    [oracle@sdbsd0 ~]$ <copy>cd sdb_demo_app</copy>
    [oracle@sdbsd0 sdb_demo_app]$ 
    ```

   

5. Start the monitoring tool via the following command. Ignore the FileNotFoundException message.

    ```
     [oracle@sdbsd0 sdb_demo_app]$ <copy>./run.sh monitor</copy>
    @oracle.monitor.Main.registerDatabase : INFO 2020-11-30T06:17:33.417 : Context : /db/demo/info
    @oracle.monitor.DatabaseMonitor$BackgroundStatusCheck.run : java.lang.ArrayIndexOutOfBoundsException : 2
    @oracle.monitor.DatabaseMonitor$BackgroundStatusCheck.run : java.lang.ArrayIndexOutOfBoundsException : 3
    java.io.FileNotFoundException: /favicon.ico
    	at oracle.monitor.FileHandler.handle(FileHandler.java:129)
    	at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:79)
    	at sun.net.httpserver.AuthFilter.doFilter(AuthFilter.java:83)
    	at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:82)
    	at sun.net.httpserver.ServerImpl$Exchange$LinkHandler.handle(ServerImpl.java:675)
    	at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:79)
    	at sun.net.httpserver.ServerImpl$Exchange.run(ServerImpl.java:647)
    	at sun.net.httpserver.ServerImpl$DefaultExecutor.execute(ServerImpl.java:158)
    	at sun.net.httpserver.ServerImpl$Dispatcher.handle(ServerImpl.java:431)
    	at sun.net.httpserver.ServerImpl$Dispatcher.run(ServerImpl.java:396)
    	at java.lang.Thread.run(Thread.java:748)
    ```

   

6. From your laptop, launch a browser and use the URL: `http://xxx.xxx.xxx.xxx:8081`. Using the public ip address of the gsm host and the port number is 8081.

    ![Demo Page](images/image-demopage.png)

   

7. Scroll down the screen, you can see the Last inserted orders:

    ![Scroll Down Page](images/image-scrolldownpage.png)

   

8. Press `Ctrl+C` to cancel the demo in both of the terminal.

    

You may now [proceed to the next lab](#next).

## Acknowledgements
* **Author** - Minqiao Wang, Aug 2024
* **Last Updated By/Date** - 