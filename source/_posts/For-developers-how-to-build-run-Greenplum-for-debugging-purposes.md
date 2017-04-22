---
title: 'For developers: how to build & run Greenplum for debugging purposes'
comments: true
date: 2017-03-08 10:32:06
tags: [数据库, Greenplum]
categories: 技术分享
---


> The binary install package downloaded from the official website is not suit for developers. Following reasons: 1) the debug symbol is often not included. 2) there is often a high level of compilier optimization. 3) you cannot match the binary executable with the original source code. As a result, you will find trouble when debugging using gdb. So, developers often dowload the source code and complile it on their own, keeping whatever infos avaliable in the binary executable for debugging. This article is about how to build Greenplum from source code and deploy it on a single machine for developer debugging purposes.

Install from source
-----------

The first step of get everything going is always the same: get the source code. The greenplum is currently hosted on github, which you can get the source code like this
```
git clone https://github.com/greenplum-db/gpdb.git
```
Afterwards, it much the same like all other Linux based software, you have to do the 3 steps: configure, make, make install.
```
./configure  CFLAGS="-O0 -gdwarf-2 -g3" --prefix=`pwd`/release  --with-perl --with-python --with-libxml --enable-mapreduce --enable-debug
make
make install
```
Here we add `-O0` to the CFLAGS, so that no complier optimization will take place. And we have `--enable-debug` configuration option set to build all the debugging info in the binary file.  We also add ` -gdwarf-2 -g3` to the CFLAGS so all the defination of  macros is keeped.
If you met configure error, it is because you are missing some of the libs refer to FAQ for more info.
After you `make install` the gp database, all the binary executable file will be located under the `release` directroy.
```
gpadmin@yshen-ThinkPad-X201:~/gpdb/release$ ls
bin  conf  demo  doc  docs  greenplum_path.sh  include  lib  sbin  share
```

Initialization
---------

### layout of the gp database cluster

It considered easier to debug  if we only have all segemets installed on one computer. By doing so, it will not cause trouble for you to set up miltiple computers or virtual machines. In the meantime, you still have the basic distribution layout of the system. So the cluster layout is as follows:

```
master host
|
----->segment host1 [primary1 primary2 primary3  mirror1 mirror2 mirror3]
```

We have master and segement hosts installed on our localhost, and 3 primary segements plus 3 mirror segements installed on segment host which is also localhost. So, alltogether we have 7 postgres backend running on localhost. You can examin how many postgres backend process is running by `ps aux | grep bin/postgres | wc -l`, in this situation, the output is 7.

### Preparation
By editing `hosts` file, the mdw ( master ) and sdw ( segment ) will point to the local host.
```
cat /etc/hosts
127.0.0.1 mdw
127.0.0.1 sdw1
```
we should also create a file `all_hosts` to keep a list of all master and segment hosts.
```
$ cat /home/gpadmin/gpdb/release/conf/all_hosts 
mdw
sdw1
```
The `seg_hosts` file only have all the segment hosts.
```
$ cat /home/gpadmin/gpdb/release/conf/seg_hosts 
sdw1
```
Later the two files will be used by `gpinitsystem` command.

We also have to set up the folder to store the data. Each primary and mirror segment plus the master segment have a folder, in our case there are 7 folders.
```
mkdir -p /home/gpadmin/gpdata/gpmaster
mkdir -p /home/gpadmin/gpdata/gpdatap1
mkdir -p /home/gpadmin/gpdata/gpdatap2
mkdir -p /home/gpadmin/gpdata/gpdatap3
mkdir -p /home/gpadmin/gpdata/gpdatam1
mkdir -p /home/gpadmin/gpdata/gpdatam2
mkdir -p /home/gpadmin/gpdata/gpdatam3
```

### Ininit the system
The file `gpinitsystem_config` is for the command `gpinitsystem` to know how many hosts are there in the cluster and how many primary/mirror segments will have on a single host. You can find a sample of configuration file in `$GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config`
```
# FILE NAME: gpinitsystem_config

# Configuration file needed by the gpinitsystem

################################################
#### REQUIRED PARAMETERS
################################################

#### Name of this Greenplum system enclosed in quotes.
ARRAY_NAME="Greenplum"

#### Naming convention for utility-generated data directories.
SEG_PREFIX=gpseg

#### Base number by which primary segment port numbers 
#### are calculated.
PORT_BASE=40000

#### File system location(s) where primary segment data directories 
#### will be created. The number of locations in the list dictate
#### the number of primary segments that will get created per
#### physical host (if multiple addresses for a host are listed in 
#### the hostfile, the number of segments will be spread evenly across
#### the specified interface addresses).
declare -a DATA_DIRECTORY=(/home/gpadmin/gpdata/gpdatap1 /home/gpadmin/gpdata/gpdatap2 /home/gpadmin/gpdata/gpdatap3)

#### OS-configured hostname or IP address of the master host.
MASTER_HOSTNAME=mdw

#### File system location where the master data directory 
#### will be created.
MASTER_DIRECTORY=/home/gpadmin/gpdata/gpmaster

#### Port number for the master instance. 
MASTER_PORT=5432

#### Shell utility used to connect to remote hosts.
TRUSTED_SHELL=ssh
#### Maximum log file segments between automatic WAL checkpoints.
CHECK_POINT_SEGMENTS=8

#### Default server-side character set encoding.
ENCODING=UNICODE

################################################
#### OPTIONAL MIRROR PARAMETERS
################################################

#### Base number by which mirror segment port numbers 
#### are calculated.
MIRROR_PORT_BASE=50000

#### Base number by which primary file replication port 
#### numbers are calculated.
REPLICATION_PORT_BASE=41000

#### Base number by which mirror file replication port 
#### numbers are calculated. 
MIRROR_REPLICATION_PORT_BASE=51000

#### File system location(s) where mirror segment data directories 
#### will be created. The number of mirror locations must equal the
#### number of primary locations as specified in the 
#### DATA_DIRECTORY parameter.
declare -a MIRROR_DATA_DIRECTORY=(/home/gpadmin/gpdata/gpdatam1 /home/gpadmin/gpdata/gpdatam2 /home/gpadmin/gpdata/gpdatam3)

################################################
#### OTHER OPTIONAL PARAMETERS
################################################

#### Create a database of this name after initialization.
DATABASE_NAME=postgres

#### Specify the location of the host address file here instead of
#### with the the -h option of gpinitsystem.
MACHINE_LIST_FILE=/home/gpadmin/gpdb/release/conf/seg_hosts

```


Before doing the initialization, we have to add a few environment variable, like `GPHOME` by doing:

```
~/gpdb/release$ source greenplum_path.sh
```
The GPHOME is set up by `configure`, so be sure to make it correct after you move the `release` folder to another location.
Inorder all the paths remain effective, you'd better add all the PATHs to your profile by:
```
cat greenplum_path.sh >> ~/.profile
```


> NOTICE: in order to run py scripts under bin directoy you have to  `export PYTHONPATH=$PYTHONPATH:~/gpdb/gpMgmt/bin/ext`

Ok, if you get here we are ready to rock and roll. Start the initialization of the system by typing the command:
```
gpadmin@yshen-ThinkPad-X201:~/gpdb/release/bin$ ./gpinitsystem -c gpinitsystem_config 
```
There will be long list of logs showing what happend.
```
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Checking configuration parameters, please wait...
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Reading Greenplum configuration file gpinitsystem_config
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Locale has not been set in gpinitsystem_config, will set to default value
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Locale set to en_US.utf8
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[WARN]:-Master hostname mdw does not match hostname output
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Checking to see if mdw can be resolved on this host
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Can resolve mdw to this host
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-MASTER_MAX_CONNECT not set, will set to default value 250
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Detected a single host GPDB array build, reducing value of BATCH_DEFAULT from 60 to 4
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[WARN]:-Master open file limit is 1024 should be >= 65535
20170307:21:30:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Checking configuration parameters, Completed
20170307:21:30:53:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Commencing multi-home checks, please wait...
.
20170307:21:30:53:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Configuring build for standard array
20170307:21:30:53:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Commencing multi-home checks, Completed
20170307:21:30:53:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Building primary segment instance array, please wait...
...
20170307:21:30:55:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Building group mirror array type , please wait...
...
20170307:21:30:57:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Checking Master host
20170307:21:30:57:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Checking new segment hosts, please wait...
20170307:21:30:58:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[WARN]:-Host mdw open files limit is 1024 should be >= 65535
......
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Checking new segment hosts, Completed
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Greenplum Database Creation Parameters
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:---------------------------------------
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master Configuration
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:---------------------------------------
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master instance name       = Greenplum
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master hostname            = mdw
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master port                = 5432
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master instance dir        = /home/gpadmin/gpdata/gpmaster/gpseg-1
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master LOCALE              = en_US.utf8
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Greenplum segment prefix   = gpseg
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master Database            = postgres
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master connections         = 250
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master buffers             = 128000kB
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Segment connections        = 750
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Segment buffers            = 128000kB
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Checkpoint segments        = 8
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Encoding                   = UNICODE
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Postgres param file        = Off
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Initdb to be used          = /home/gpadmin/gpdb/release/bin/initdb
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-GP_LIBRARY_PATH is         = /home/gpadmin/gpdb/release/lib
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[WARN]:-Ulimit check               = Warnings generated, see log file <<<<<
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Array host connect type    = Single hostname per node
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master IP address [1]      = ::1
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master IP address [2]      = 192.168.1.111
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master IP address [3]      = fe80::e181:909d:d2a4:8de0
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Standby Master             = Not Configured
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Primary segment #          = 3
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Total Database segments    = 3
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Trusted shell              = ssh
20170307:21:31:04:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Number segment hosts       = 1
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Mirror port base           = 50000
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Replicaton port base       = 41000
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Mirror replicaton port base= 51000
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Mirror segment #           = 3
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Mirroring config           = ON
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Mirroring type             = Group
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:----------------------------------------
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Greenplum Primary Segment Configuration
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:----------------------------------------
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-sdw1 	/home/gpadmin/gpdata/gpdatap1/gpseg0 	40000 	2 	0 	41000
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-sdw1 	/home/gpadmin/gpdata/gpdatap2/gpseg1 	40001 	3 	1 	41001
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-sdw1 	/home/gpadmin/gpdata/gpdatap3/gpseg2 	40002 	4 	2 	41002
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:---------------------------------------
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Greenplum Mirror Segment Configuration
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:---------------------------------------
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-sdw1 	/home/gpadmin/gpdata/gpdatam1/gpseg0 	50000 	5 	0 	51000
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-sdw1 	/home/gpadmin/gpdata/gpdatam2/gpseg1 	50001 	6 	1 	51001
20170307:21:31:05:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-sdw1 	/home/gpadmin/gpdata/gpdatam3/gpseg2 	50002 	7 	2 	51002

Continue with Greenplum creation Yy|Nn (default=N):
> Y
20170307:21:31:07:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Building the Master instance database, please wait...
20170307:21:31:15:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Starting the Master in admin mode
20170307:21:31:23:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Commencing parallel build of primary segment instances
20170307:21:31:23:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Spawning parallel processes    batch [1], please wait...
...
20170307:21:31:24:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Waiting for parallel processes batch [1], please wait...
............................
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:------------------------------------------------
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Parallel process exit status
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:------------------------------------------------
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Total processes marked as completed           = 3
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Total processes marked as killed              = 0
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Total processes marked as failed              = 0
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:------------------------------------------------
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Commencing parallel build of mirror segment instances
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Spawning parallel processes    batch [1], please wait...
...
20170307:21:31:52:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Waiting for parallel processes batch [1], please wait...
...................
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:------------------------------------------------
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Parallel process exit status
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:------------------------------------------------
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Total processes marked as completed           = 3
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Total processes marked as killed              = 0
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Total processes marked as failed              = 0
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:------------------------------------------------
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Deleting distributed backout files
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Removing back out file
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-No errors generated from parallel processes
20170307:21:32:12:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Restarting the Greenplum instance in production mode
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Starting gpstop with args: -a -l /home/gpadmin/gpAdminLogs -i -m -d /home/gpadmin/gpdata/gpmaster/gpseg-1
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Gathering information and validating the environment...
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Obtaining Segment details from master...
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Greenplum Version: 'postgres (Greenplum Database) 5.0.0-alpha.0-96-g0701a77 build dev'
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-There are 0 connections to the database
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Commencing Master instance shutdown with mode='immediate'
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master host=yshen-ThinkPad-X201
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Commencing Master instance shutdown with mode=immediate
20170307:21:32:13:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master segment instance directory=/home/gpadmin/gpdata/gpmaster/gpseg-1
20170307:21:32:14:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Attempting forceful termination of any leftover master process
20170307:21:32:14:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[INFO]:-Terminating processes for segment /home/gpadmin/gpdata/gpmaster/gpseg-1
20170307:21:32:14:004724 gpstop:yshen-ThinkPad-X201:gpadmin-[ERROR]:-Failed to kill processes for segment /home/gpadmin/gpdata/gpmaster/gpseg-1: ([Errno 3] No such process)
20170307:21:32:14:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Starting gpstart with args: -a -l /home/gpadmin/gpAdminLogs -d /home/gpadmin/gpdata/gpmaster/gpseg-1
20170307:21:32:14:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Gathering information and validating the environment...
20170307:21:32:14:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Greenplum Binary Version: 'postgres (Greenplum Database) 5.0.0-alpha.0-96-g0701a77 build dev'
20170307:21:32:14:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Greenplum Catalog Version: '301702171'
20170307:21:32:14:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Starting Master instance in admin mode
20170307:21:32:15:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20170307:21:32:15:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Obtaining Segment details from master...
20170307:21:32:15:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Setting new master era
20170307:21:32:15:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Master Started...
20170307:21:32:15:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Shutting down master
20170307:21:32:17:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Commencing parallel primary and mirror segment instance startup, please wait...
... 
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Process results...
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-----------------------------------------------------
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-   Successful segment starts                                            = 6
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-   Failed segment starts                                                = 0
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-   Skipped segment starts (segments are marked down in configuration)   = 0
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-----------------------------------------------------
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Successfully started 6 of 6 segment instances 
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-----------------------------------------------------
20170307:21:32:20:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Starting Master instance yshen-ThinkPad-X201 directory /home/gpadmin/gpdata/gpmaster/gpseg-1 
20170307:21:32:21:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Command pg_ctl reports Master yshen-ThinkPad-X201 instance active
20170307:21:32:21:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-No standby master configured.  skipping...
20170307:21:32:21:004811 gpstart:yshen-ThinkPad-X201:gpadmin-[INFO]:-Database successfully started
20170307:21:32:21:010106 gpinitsystem:yshen-ThinkPad-X201:gpadmin-[INFO]:-Completed restart of Greenplum instance in production mode
20170307:21:32:21:gpinitsystem:yshen-ThinkPad-X201:gpadmin-[FATAL]:-Failed to complete create database postgres  Script Exiting!

```


startup & stop database server
------------
There are python script `gpstart`\`gpstop` to start and stop the cluster all at once, so will save your the trouble of start\stop each of the segment by yourself.

```
gpadmin@yshen-ThinkPad-X201:~/gpdb/release/bin$ ./gpstart -d ~/gpdata/gpmaster/gpseg-1/
```
When the cluster is up and running, you can see each segment is running under each's data directory. And you can tell easily which of them is master segment and primary/mirror segment.
```

$ ps aux | grep bin/postgres
gpadmin   4938  0.3  2.8 426780 230960 ?       Ss   21:32   0:00 /home/gpadmin/gpdb/release/bin/postgres -D /home/gpadmin/gpdata/gpdatap2/gpseg1 -p 40001 --gp_dbid=3 --gp_num_contents_in_cluster=3 --silent-mode=true -i -M quiescent --gp_contentid=1
gpadmin   4939  0.3  2.8 424808 229148 ?       Ss   21:32   0:00 /home/gpadmin/gpdb/release/bin/postgres -D /home/gpadmin/gpdata/gpdatam2/gpseg1 -p 50001 --gp_dbid=6 --gp_num_contents_in_cluster=3 --silent-mode=true -i -M quiescent --gp_contentid=1
gpadmin   4944  0.3  2.8 424804 229172 ?       Ss   21:32   0:00 /home/gpadmin/gpdb/release/bin/postgres -D /home/gpadmin/gpdata/gpdatam1/gpseg0 -p 50000 --gp_dbid=5 --gp_num_contents_in_cluster=3 --silent-mode=true -i -M quiescent --gp_contentid=0
gpadmin   4946  0.3  2.9 426784 231048 ?       Ss   21:32   0:00 /home/gpadmin/gpdb/release/bin/postgres -D /home/gpadmin/gpdata/gpdatap3/gpseg2 -p 40002 --gp_dbid=4 --gp_num_contents_in_cluster=3 --silent-mode=true -i -M quiescent --gp_contentid=2
gpadmin   4947  0.3  2.8 424808 229128 ?       Ss   21:32   0:00 /home/gpadmin/gpdb/release/bin/postgres -D /home/gpadmin/gpdata/gpdatam3/gpseg2 -p 50002 --gp_dbid=7 --gp_num_contents_in_cluster=3 --silent-mode=true -i -M quiescent --gp_contentid=2
gpadmin   4948  0.3  2.8 426784 230904 ?       Ss   21:32   0:00 /home/gpadmin/gpdb/release/bin/postgres -D /home/gpadmin/gpdata/gpdatap1/gpseg0 -p 40000 --gp_dbid=2 --gp_num_contents_in_cluster=3 --silent-mode=true -i -M quiescent --gp_contentid=0
gpadmin   5039  0.3  2.5 384580 205840 ?       Ss   21:32   0:00 /home/gpadmin/gpdb/release/bin/postgres -D /home/gpadmin/gpdata/gpmaster/gpseg-1 -p 5432 --gp_dbid=1 --gp_num_contents_in_cluster=3 --silent-mode=true -i -M master --gp_contentid=-1 -x 0 -E
gpadmin   5306  0.0  0.0  21312  1092 pts/1    S+   21:33   0:00 grep --color=auto bin/postgres

```
Much the same like how to start the cluster, you can use python script `gpstop` to stop all the cluster at once.
```
gpadmin@yshen-ThinkPad-X201:~/gpdb/release/bin$ ./gpstop -d ~/gpdata/gpmaster/gpseg-1/
```


run regression test
--------------------
The first time cluster is running you can do a regression test to varify the correctness of your code.
```
make installcheck-world 
```
It took quite a while to finish all the tests. Later whenever you modify the source code you will have to run the regression test to make sure you do not affect other part of the code.

Connecting to greenplum cluster using psql
--------------
You can connect to the `master segment` by  a CLI tool  called `psql`. The default port is `5432` and the default database we connect to is `postgres`.
```
gpadmin@yshen-ThinkPad-X201:~/gpdb/release/bin$ ./psql
psql (8.3.23)
Type "help" for help.

postgres=# SELECT version();
                                                                                                version                                     
                                                            
--------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------
 PostgreSQL 8.3.23 (Greenplum Database 5.0.0-alpha.0-96-g0701a77 build dev) on x86_64-pc-linux-gnu, compiled by GCC gcc (Ubuntu 5.4.0-6ubunt
u1~16.04.4) 5.4.0 20160609 compiled on Mar  7 2017 18:25:13
(1 row)

```

You can issue a query to find out how the cluster is configured by
```

postgres=# SELECT * from gp_segment_configuration order by 1;
 dbid | content | role | preferred_role | mode | status | port  |      hostname       | address | replication_port | san_mounts 
------+---------+------+----------------+------+--------+-------+---------------------+---------+------------------+------------
    1 |      -1 | p    | p              | s    | u      |  5432 | yshen-ThinkPad-X201 | mdw     |                  | 
    2 |       0 | m    | p              | s    | d      | 40000 | yshen-ThinkPad-X201 | sdw1    |            41000 | 
    3 |       1 | p    | p              | s    | u      | 40001 | yshen-ThinkPad-X201 | sdw1    |            41001 | 
    4 |       2 | p    | p              | s    | u      | 40002 | yshen-ThinkPad-X201 | sdw1    |            41002 | 
    5 |       0 | p    | m              | c    | u      | 50000 | yshen-ThinkPad-X201 | sdw1    |            51000 | 
    6 |       1 | m    | m              | s    | u      | 50001 | yshen-ThinkPad-X201 | sdw1    |            51001 | 
    7 |       2 | m    | m              | s    | u      | 50002 | yshen-ThinkPad-X201 | sdw1    |            51002 | 
(7 rows)

```
Here is some explaination about each column.

dbid : range from 1-N, where master segment is 1, then the primary segment, the last is the mirror segment.

content : master is -1, each primary & mirror segment share the same value since there content is the same.

role : primary or mirror status of a segment is running in.

preferred_role : the original defination of primary or mirror for a segment.

mode : 's' means synchronized. 'r' means resyncing. 'c' change logging.

status : 'u' means up. 'd' means down.



Feature testing
--------
Different from the original postgres, greenplum allows you to store your table data separetly on each segment host. The master segment stores only meta-data while each primary/mirror segment store the acutal data.
You can setup the distribution key when creating table:
```
postgres=# CREATE TABLE tb1 ( id int, name varchar(100)) DISTRIBUTED BY (id);
CREATE TABLE
postgres=# \d tb1
             Table "public.tb1"
 Column |          Type          | Modifiers 
--------+------------------------+-----------
 id     | integer                | 
 name   | character varying(100) | 
Distributed by: (id)


postgres=# INSERT INTO tb1 values(generate_series(1,1000));
INSERT 0 1000
```
After the data is inserted all the data is distribute by ID on each segment host. You can exmain how they are distributed by issuing command:
```
postgres=# select gp_segment_id,count(*) from tb1 group by 1 order by 1;
 gp_segment_id | count 
---------------+-------
             0 |   321
             1 |   342
             2 |   337
(3 rows)

```

Or you can find out which segment a specific tuple is reside by the system column `gp_segment_id` :

```
postgres=# SELECT gp_segment_id , * from tb1 where id = 99;
 gp_segment_id | id |       name        
---------------+----+-------------------
             2 | 99 | 0.459008027799428
(1 row)
```

When you query some data from the cluster. The plan is parallized, each segment is execute a seq scan on its part of data and gathered by the master host

```
postgres=# EXPLAIN SELECT * FROM tb1;
                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Gather Motion 3:1  (slice1; segments: 3)  (cost=0.00..13.00 rows=1000 width=22)
   ->  Seq Scan on tb1  (cost=0.00..13.00 rows=334 width=22)
(2 rows)
```

FAQ
------

### dependences might have to be installed
```
sudo apt install libapr1-dev   libevent-dev 
sudo apt install  libxml2*
sudo apt install libcurl3-dev
sudo apt install libghc-bzlib-dev
sudo apt install libyaml-dev
sudo apt install libpython-all-dev
sudo apt install libperl-dev
sudo apt install libpython-all-dev
sudo apt install python-py  pip
```


### cannot establish ssh access into the local host
if you see the following erro message:
```
$ gpssh-exkeys -f ../conf/all_hosts 
[STEP 1 of 5] create local ID and authorize on local host
  ... /home/yshen/.ssh/id_rsa file exists ... key generation skipped
[ERROR yshen-ThinkPad-X201] authentication check failed:
     ssh: connect to host yshen-thinkpad-x201 port 22: Connection refused
[ERROR] cannot establish ssh access into the local host

```
that means you have't install ssh-server on your os, run the following commands to do so:
```
$  sudo apt-get install openssh-server
$ sudo /etc/init.d/ssh start
[ ok ] Starting ssh (via systemctl): ssh.service.

```

###  bad password

```
$ gpssh-exkeys -f ../conf/all_hosts 
[STEP 1 of 5] create local ID and authorize on local host
  ... /home/yshen/.ssh/id_rsa file exists ... key generation skipped

[STEP 2 of 5] keyscan all hosts and update known_hosts file

[STEP 3 of 5] authorize current user on remote hosts
  ... send to mdw
  ***
  *** Enter password for mdw: 
[ERROR mdw] bad password
  ***
  *** Enter password for mdw: 
[ERROR mdw] bad password
  ***
  *** Enter password for mdw: 
[ERROR mdw] bad password
  ***
  *** Enter password for mdw: 
```
If you met the above error message, means you are not logged in as gpadmin user.

### No module named paramiko
```
$ ./gpssh-exkeys -f ../conf/all_hosts 
Error: unable to import module: No module named paramiko
```
You are missing some of the python libs.
```
pip install paramiko psutil
```


### cannot run the python script under $GPHOME/bin dir

```
~/gpdb/release/bin$ export PYTHONPATH=~/gpdb/gpMgmt/bin/ext:$PYTHONPATH
```

### ImportError: No module named lockfile.pidlockfile
```
$ ./gpstart -a
Traceback (most recent call last):
  File "./gpstart", line 9, in <module>
    from gppylib.mainUtils import *
  File "/home/gpadmin/gpdb/release/lib/python/gppylib/mainUtils.py", line 37, in <module>
    from lockfile.pidlockfile import PIDLockFile, LockTimeout
ImportError: No module named lockfile.pidlockfile
```You are missing some of the python libs.
You are missing some of the python libs.
```
pip install lockfile
```

