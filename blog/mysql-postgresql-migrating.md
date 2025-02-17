---
title: Migrating from MySQL to PostgreSQL for IceData
date: 2025-02-15
credits: ovler
---

# Migrating from MySQL to PostgreSQL for IceData

## Basic Info

50M rows, 3.1GB, from MySQL 8.0 to PostgreSQL 17

## replica 

The first tool I tried is the famous pgloader. But it have so many minor problems:

### Condition QMYND:MYSQL-UNSUPPORTED-AUTHENTICATION

Solve: https://stackoverflow.com/questions/56542036/pgloader-failed-to-connect-to-mysql-at-localhost-port-3306-as-user-root

> The problem is that currently *pgloader* doesn't support `caching_sha2_password` authentication plugin, which is default for MySQL 8, whereas older MySQL versions use `mysql_native_password` plugin.
>
> Based on [this comment](https://github.com/dimitri/pgloader/issues/782#issuecomment-502323324), the workaround here is to edit `my.cnf` (if you don't know where it is, look [here](https://stackoverflow.com/a/9603176/6609485)) and in `[mysqld]` section add
>
> ```ini
> default-authentication-plugin=mysql_native_password
> ```
>
> Then restart your MySQL server and execute:
>
> ```sql
> ALTER USER 'youruser'@'localhost' IDENTIFIED WITH mysql_native_password BY 'yourpassword';
> ```
>
> After that the error must be gone.

Then, trying to edit `my.cnf`(actually `my.ini`) on the Windows machine of the replica MySQL using windows notepad, the save of file breaks the encoding of that file——From `ANSI`to `utf8`, which caused crash loop.

> ```mysql
> Found option without preceding group in config file: /etc/mysql/my.cnf at line: 1
> ```

Gathering the information from [SO](https://stackoverflow.com/questions/8020297/mysql-my-cnf-file-found-option-without-preceding-group), the automatic `utf8` convention is the root of the crash loop.

After fixing it, yet another bug occurs:

Trying to run

```
ALTER USER 'jack'@'localhost' IDENTIFIED WITH mysql_native_password BY 'yourpassword';
```

But got

```mysql
ERROR 1396 (HY000): Operation ALTER USER failed for 'jack'@'localhost'
```

This bug has been sitting on bugs.mysql.com since 2007, and it seems that people on [SO](https://stackoverflow.com/questions/5555328/error-1396-hy000-operation-create-user-failed-for-jacklocalhost) have different opinions. So I just create new users for sync.

```mysql
CREATE USER 'app_sync'@'localhost' IDENTIFIED WITH mysql_native_password BY 'some_random_pass';

GRANT ALL ON *.* TO 'app_sync2'@'localhost' WITH GRANT OPTION;
GRANT RELOAD ON *.* TO 'app_sync2'@'localhost' WITH GRANT OPTION;
GRANT REPLICATION CLIENT ON *.* TO 'app_sync2'@'localhost' WITH GRANT OPTION;
GRANT REPLICATION SLAVE ON *.* TO 'app_sync2'@'localhost' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

Then it works.

### ERROR mysql: 76 fell through ECASE expression.

```shell
pgloader mysql://jack:REDACTED@REDACTED:3306/hantang pgsql://REDACTED@127.0.0.1:5432/icedata
2025-02-16T10:35:06.036000Z LOG pgloader version "3.6.10~devel"
2025-02-16T10:35:06.044000Z LOG Data errors in '/tmp/pgloader/'
2025-02-16T10:35:06.176002Z LOG Migrating from #<MYSQL-CONNECTION mysql://app_sync2@REDACTED:3306/hantang {REDACTED}>
2025-02-16T10:35:06.176002Z LOG Migrating into #<PGSQL-CONNECTION pgsql://dbuser_dba@127.0.0.1:5432/icedata {REDACTED}>
2025-02-16T10:35:06.416004Z ERROR mysql: 76 fell through ECASE expression.
       Wanted one of (2 3 4 5 6 8 9 10 11 14 15 17 20 21 23 27 28 30 31 32 33
                      35 41 42 45 46 47 48 49 50 51 52 54 55 56 60 61 62 63 64
                      65 69 72 77 78 79 82 83 87 90 92 93 94 95 96 97 98 101
                      102 103 104 105 106 107 108 109 110 111 112 113 114 115
                      116 117 118 119 120 121 122 123 124 128 129 130 131 132
                      133 134 135 136 137 138 139 140 141 142 143 144 145 146
                      147 148 149 150 151 159 160 161 162 163 164 165 166 167
                      168 169 170 171 172 173 174 175 176 177 178 179 180 181
                      182 183 192 193 194 195 196 197 198 199 200 201 202 203
                      204 205 206 207 208 209 210 211 212 213 214 215 223 224
                      225 226 227 228 229 230 231 232 233 234 235 236 237 238
                      239 240 241 242 243 244 245 246 247 254).
2025-02-16T10:35:06.416004Z LOG report summary reset
       table name     errors       rows      bytes      total time
-----------------  ---------  ---------  ---------  --------------
  fetch meta data          0          0                     0.000s
-----------------  ---------  ---------  ---------  --------------
-----------------  ---------  ---------  ---------  --------------
```

Yet another bug of pgloader.

Following guide here on [Stake Overflow](https://stackoverflow.com/questions/75493241/mysql-to-postgres-migration-pgloader-error-mysql-76-fell-through-ecase-express), we need to build the pgloader by ourself.

> ```shell
> apt remove pgloader -y
> git clone https://github.com/dimitri/pgloader.git
> apt-get install sbcl unzip libsqlite3-dev make curl gawk freetds-dev libzip-dev
> cd pgloader
> make pgloader
> ./build/bin/pgloader --help
> ./build/bin/pgloader mysql://root:'123'@127.0.0.1/nextcloud_storage postgresql://postgres:pass@127.0.0.1:5432/nextcloud_storage
> ```

It worked. But the build is so time-consuming, and the bug is known since 2019. No Fix.

### Run the sync

Finally, we can run the sync. It took me 2h30m5.174s to sync the 3.1GB database.

```shell
build/bin/pgloader mysql://jack:REDACTED@REDACTED:3306/hantang pgsql://REDACTED@127.0.0.1:5432/icedata
2025-02-16T10:35:06.036000Z LOG pgloader version "3.6.10~devel"
2025-02-16T10:35:06.044000Z LOG Data errors in '/tmp/pgloader/'
2025-02-16T10:35:06.176002Z LOG Migrating from #<MYSQL-CONNECTION mysql://app_sync2@REDACTED:3306/hantang {REDACTED}>
2025-02-16T10:35:06.176002Z LOG Migrating into #<PGSQL-CONNECTION pgsql://dbuser_dba@127.0.0.1:5432/icedata {REDACTED}>
2025-02-16T21:10:26.714403+08:00 ERROR PostgreSQL Database error 0A000: access method "gin" does not support unique indexes
QUERY: CREATE UNIQUE INDEX idx_23028_identifier ON hantang.identifier_map USING gin(identifier);
2025-02-16T21:12:19.711468+08:00 LOG report summary reset
                  table name     errors       rows      bytes      total time
----------------------------  ---------  ---------  ---------  --------------
             fetch meta data          0         39                     0.436s
              Create Schemas          0          0                     0.004s
            Create SQL Types          0          0                     0.008s
               Create tables          0         22                     0.040s
              Set Table OIDs          0         11                     0.012s
----------------------------  ---------  ---------  ---------  --------------
         hantang.video_daily          0   27099896     1.5 GB     2h7m22.629s
        hantang.video_record          0   18386435     1.2 GB    1h52m39.561s
        hantang.video_minute          0    4685170   353.1 MB      25m52.027s
        hantang.video_static          0     281249    96.4 MB       9m28.389s
      hantang.identifier_map          0        691    18.6 kB          0.268s
              hantang.qq_map          0         49     1.1 kB          0.196s
hantang.olap_rel_video_vocal          0          0                     0.192s
            hantang.dim_user          0      53921     4.8 MB         12.616s
            hantang.dim_type          0        118     1.8 kB          0.232s
           hantang.task_list          0         25     2.5 kB          0.204s
      hantang.video_record_1          0          0                     0.224s
----------------------------  ---------  ---------  ---------  --------------
     COPY Threads Completion          0          4               2h18m44.647s
              Create Indexes          1         27                 11m20.342s
      Index Build Completion          0         28                     0.036s
             Reset Sequences          0          2                     0.088s
                Primary Keys          0          9                     0.016s
         Create Foreign Keys          0          0                     0.000s
             Create Triggers          0          0                     0.000s
             Set Search Path          0          1                     0.004s
            Install Comments          0         43                     0.040s
----------------------------  ---------  ---------  ---------  --------------
           Total import time          ✓   50507554     3.1 GB     2h30m5.174s
```

### Unable to Incremental sync

While running the sync, there are new data every minute. But pgloader don't support Incremental sync. What the ……

I'm now looking into [pg_chameleon](https://github.com/the4thdoctor/pg_chameleon), I hope this new tool can give me some relief.

## pg_chameleon

It still have some problems.

### Speed

First is speed. It was started at 2025-02-17 03:16:57, end at 2025-02-17 06:27:33, used more than 3 hours. And the memory pressure is higher.

### Incremental sync

I enabled the binlog after the full sync, causing data between full sync and enabled binlog unable to be synced.

### Outdated Document

https://github.com/the4thdoctor/pg_chameleon/issues/177

I noticed that the documentation on [www.pgchameleon.org](https://www.pgchameleon.org) is outdated compared to other sources. The website references version **v2.0.19**, while the documentation on [ReadTheDocs](https://pg-chameleon.readthedocs.io/en/main/) is for **v2.0.20**—but it seems like version **v2.0.21** was actually released last month ([January 21, 2025](https://github.com/the4thdoctor/pg_chameleon/releases/tag/v2.0.21)).  

This discrepancy caused some confusion when I was setting up my configuration because there are differences between the guides.  

For example, in the usage.html, `Add the configuration for the replica to my.cnf` section:

The guide on [pgchameleon's](https://pgchameleon.org/documents/usage.html) website does not include this important line from [ReadTheDocs](https://pg-chameleon.readthedocs.io/en/main/usage.html):

``` conf
# MARIADB 10.5.0+ OR MYSQL 8.0.14+ versions
binlog_row_metadata = FULL
```

This omission led me to accidentally skip that setting during setup.

### Killing Loop

https://github.com/the4thdoctor/pg_chameleon/blob/5458575165565b4593d33af912e7fdbeb4db49f8/pg_chameleon/lib/global_lib.py#L699-L703

```python
        replica_pid = os.path.expanduser('%s/%s.pid' % (self.config["pid_dir"],self.source))
        if os.path.isfile(replica_pid):
            try:
                file_pid=open(replica_pid,'r')
                pid=file_pid.read()
                file_pid.close()
                os.kill(int(pid),2)
                print("Requesting the replica for source %s to stop" % (self.source))
                while True:
                    try:
                        os.kill(int(pid),0)
                    except:
                        break
                print("The replica process is stopped")
            except:
                print("An error occurred when trying to signal the replica process")
```

Oh…… Does this code really know what they're doing?

The thing happened on my machine is, that process don't eat the kill 0. It only kills when kill 9. So it's in a loop, killing and killing, causing 100% CPU on that core. It's lucky that this is not Multi-threaded!

Kill by shell solves this problem. Perhaps they should add some limit.

Working in progress! maybe updated, maybe not.
