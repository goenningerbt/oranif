# oranif
[![Build Status](https://travis-ci.org/KonnexionsGmbH/oranif.svg?branch=master)](https://travis-ci.org/KonnexionsGmbH/oranif) [![Coverage Status](https://coveralls.io/repos/github/KonnexionsGmbH/oranif/badge.svg?branch=master)](https://coveralls.io/github/KonnexionsGmbH/oranif?branch=master)
![GitHub](https://img.shields.io/github/license/KonnexionsGmbH/oranif.svg)

Oracle Call Interface driver using dirty NIFs. Requires Erlang/OTP 20 or later with full dirty nif support.

## Development
Currently builds in Window, Linux and OS X

## Compile (all OSs)
```sh
# embed odpic source (default)
rebar3 compile
# link with compiled odpic library
# (c_src/odpi/lib/ needed at runtime for NIF load)
LINKODPI=true rebar3 compile
ORANIF_DEBUG=_verbosity_ rebar3 compile # debug log verbosity >= 1
# see dpi_nif.h for ORANIF_DEBUG values and debug log granularities
```

### OSX/Linux
- Requires Oracle Client library installed, see https://oracle.github.io/odpi/doc/installation.html for installation instructions.
- For OSX use `basic` as `basic-lite` didn't work in our tests.
- Requires a C compiler supporting the c11 standard.
- code coverage
```sh
gcov -o ./ c_src/*.c
```

### Ubuntu (Windows Subsystem for Linux)
```sh
$ uname -a
Linux WKS006 4.4.0-17763-Microsoft #379-Microsoft Wed Mar 06 19:16:00 PST 2019 x86_64 x86_64 x86_64 GNU/Linux
$ sudo apt-get install libaio1
$ export LD_LIBRARY_PATH=$ROOT/oranif/c_src/odpi/lib/
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/oracle/19.3/client64/lib/ # or `sudo ldconfig`
```
#### code coverage setup (first time)
```sh
$ wget http://ftp.de.debian.org/debian/pool/main/l/lcov/lcov_1.11.orig.tar.gz
$ tar xf lcov_1.11.orig.tar.gz
$ sudo make -C lcov-1.11/ install
```
#### code coverage report
```sh
lcov --directory . --capture --output-file coverage.info
lcov --list coverage.info
```
_A `cover.sh [target_percentage]` script is also provided for coverage check after `rebar3 eunit`._

# Testing
## Setting up docker container
```sh
[HOST] docker create --shm-size 1G -p 1521:1521/t cp -e ORACLE_PWD=oracle konnexionsgmbh/db_19_3_ee
Change 0.0.0.0 to a system interface IP (ipconfig / ifconfig) below
[CONTAINER/HOST] sqlplus sys/oracle@0.0.0.0:1521/orclpdb1 as sysdba @test/db_setup.sql $(pwd)/log/
```

`rebar3 eunit` (with `tests/connect.config` configuration). And for testing with code coverage using lcov `rebar3 as lcov eunit`

## DB Init SQL (XE)
```cmd
C:\> sqlplus system
```
```sql
EXEC DBMS_XDB.SETLISTENERLOCALACCESS(FALSE);
alter session set "_ORACLE_SCRIPT"=true;

create user scott identified by tiger;

grant alter system to scott;
grant create session to scott;
grant unlimited tablespace to scott;
grant create table to scott;
grant create cluster to scott;
grant create view to scott;
grant create sequence to scott;
grant create procedure to scott;
grant create trigger to scott;
grant create any directory to scott;
grant drop any directory to scott;
grant create type to scott;
grant create operator to scott;
grant create indextype to scott;

exit
```
```cmd
C:\> sqlplus scott/tiger@127.0.0.1:1521/xe
```
```sql
select * from session_privs;
```

## Increase amount of processes, sessions and transactions
Extensive database usage may exhaust the connection limits that have defaults around 100. If that happens, any attempts to establish further connections may fail, resulting in an error such as "ORA-12516: TNS:listener could not find available handler with matching protocol stack". To prevent this, the limit should be raised so the database can establish more simultaneous connections.

To do this, log into the database as SYSDBA. Normal users aren't privileged enough.

```
sqlplus sys as sysdba
```

Then, set the new limits:

```
alter system set processes=1000 scope=spfile
alter system set sessions=1000 scope=spfile
alter system set transactions=1000 scope=spfile
```

In this example, the limits are set to 1000. The actual values might be even higher because there are constraints regarding the size of those numbers. For instance, "sessions" might be set to 1524 instead.

Then, the database must be restarted for this change to take effect:

```
Windows Menu --> Oracle Database 11g Express Edition --> Stop database
```

This takes a while, as indicated by the console window that opens. When it is done, start the database:

```
Windows Menu --> Oracle Database 11g Express Edition --> Start database
```
Verify that the change took effect using sqlplus:

```
show parameter sessions 
```

A result similar to this should be printed:

```
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
java_max_sessionspace_size           integer     0
java_soft_sessionspace_limit         integer     0
license_max_sessions                 integer     0
license_sessions_warning             integer     0
sessions                             integer     1524
shared_server_sessions               integer
```
