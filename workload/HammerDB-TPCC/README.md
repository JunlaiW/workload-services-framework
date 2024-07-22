>
> **Note: The Workload Services Framework is a benchmarking framework and is not intended to be used for the deployment of workloads in production environments. It is recommended that users consider any adjustments which may be necessary for the deployment of these workloads in a production environment including those necessary for implementing software best practices for workload scalability and security.**
>
### Introduction

HammerDB is the leading benchmarking and load testing software for the worlds most popular databases supporting Oracle Database, SQL Server, IBM Db2, MySQL, MariaDB and PostgreSQL.

This workload uses HammerDB to measure Database(s) performance. At this moment, this benchmarks measure performance of these databases :

* MySQL


### Test Case
Below are the list of testcase(s) for specific database(s).

#### MySQL

There are currently testcases that measure MySQL performance. 

* `test_static_hammerdb_tpcc_mysql8031_base_disk_hugepage_off_pkm` - PKM Testcase
    1. This testcase is used for measuring performace, and same with `test_static_hammerdb_tpcc_mysql8031_base_disk_hugepage_off`
* `test_static_hammerdb_tpcc_mysql8031_base_disk_hugepage_off_gated` - Gated Testcase
    1. This testcase is only used for functional test
* `test_static_hammerdb_tpcc_mysql8031_base_disk_hugepage_on` - Hugepage on Testcase
    1. This testcase is hugepage on case

For performance evaluation, `test_static_hammerdb_tpcc_mysql8031_base_disk_hugepage_off_pkm` is recommanded. This workload supports single node test(MySQL and HammerDB on same node but different socket/NUMA node) and two nodes test(MySQL and HammerDB on different node). 

### Docker Image

The docker image for this workload is:
* `tpcc-hammerdb` - Docker image to run benchmark with HammerDB v4.4.
* `tpcc-mysql` - Docker image of database with MySQL 8.0.31.
    * required for all testcase:
        * This image/pod should be executed on Kubernetes node with `HAS-SETUP-DISK-MOUNT-1="yes"` label.
    * required for hugepage on testcase:
        * This image/pod should be executed on kubernetes node with `HAS-SETUP-HUGEPAGE-2048kB-<your_hugepage_size>` label

### Prerequisite
#### Setup the SUT(required for all testcases)
Setup the SUT(target BMs) with [setup-sut-k8s.sh](/script/setup/setup-sut-k8s.sh). This should create a k8s cluster with SUT. [Document](/doc/user-guide/preparing-infrastructure/setup-wsf.md#setup-sut-k8ssh) for this script.

You need to check if the cluster is healthy and make sure that there is no label `HAS-SETUP-DISK-MOUNT-1=yes` on client and controller node.

#### Set up data disk(required for all testcases)
Data disk should be mounted to `/mnt/disk1` and an NVME drive is recommended for optimal DB performance. 

Set up disk if running disk test case, below is the script for reference:
```shell
device="/dev/nvme0n1"
partition="/dev/nvme0n1p1"
mntDir="/mnt/disk1"

sudo apt-get update
sudo apt-get install parted
sudo parted $device mklabel gpt
sudo parted -a opt $device mkpart primary xfs 0% 100%

sudo mkfs.xfs -L datapartition $partition
sudo mkdir -p $mntDir
sudo mount -o defaults $partition $mntDir

lsblk --fs
cp /etc/fstab /etc/fstab.bak
echo "LABEL=datapartition /mnt/disk1 xfs defaults 0 2" >> /etc/fstab
```

#### Set up hugepage(only required for hugepage on testcase)
```
# configure 32GB hugepages by default
sudo grubby --update-kernel=DEFAULT --args="hugepages=16384"

# configure kubernetes with HAS-SETUP-HUGEPAGE-2048kB-16384 label
kubectl label nodes <your_node_name> HAS-SETUP-HUGEPAGE-2048kB-16384="yes"

# reboot is REQUIRED

# once testcase is completed, you can remove hugepages configuration & the label
sudo grubby --update-kernel=DEFAULT --remove-args="hugepages=16384"
kubectl label nodes <your_node_name> HAS-SETUP-HUGEPAGE-2048kB-16384-

# The hugepage size depends on buffer pool size of database, by default the ratio is 0.5, for example:

Database    Buffer pool size    Hugepage size         Hugepages                      Label               
mysql               96GB                128GB           65536             HAS-SETUP-HUGEPAGE-2048kB-32768

```

### How to run the workload
#### MySQL Test Major Parameters
Below are the parameters REQUIRED for this workload, you need to specify these parameters in the test config file or using `./ctest.sh --set`:
1. TPCC_NUM_WAREHOUSES: number of warehouses in HammerDB. 
2. DB_CPU_LIMIT: number of cpus used by MySQL, should equal to #vcpus that mysql will use.
3. TPCC_HAMMER_NUM_VIRTUAL_USERS: 
    - For single instance, first scaling with different VU(4_5_6_7_8_9_10_11_12_13_14) and then run 3 times with fixed VU(13).
4. ENABLE_SOCKET_BIND, SERVER_SOCKET_BIND_NODE, CLIENT_SOCKET_BIND_NODE: Need to set when socket bind is needed. By default, `ENABLE_SOCKET_BIND=false`, `SERVER_SOCKET_BIND_NODE=''`, `CLIENT_SOCKET_BIND_NODE=''`.
5. RUN_SINGLE_NODE: Need to set when server and client on same node.

#### Confirm your test config
For performance evaluation, `test_static_hammerdb_tpcc_mysql8031_base_disk_hugepage_off_pkm` is recommanded. This workload supports single node test(MySQL and HammerDB on same node but different socket/NUMA node) and two nodes test(MySQL and HammerDB on different node). 

Test configs for different scenories:
1. Two node test with all NUMA nodes used:
```
TPCC_NUM_WAREHOUSES: 2000
DB_CPU_LIMIT: 128
#TPCC_HAMMER_NUM_VIRTUAL_USERS: "224_240_256_272_288" # First run VU scaling and find the peak VU
TPCC_HAMMER_NUM_VIRTUAL_USERS: "256" # Second run with fixed VU(peak VU found in VU scaling) for serveral times

# comment out since we are using default value
#ENABLE_SOCKET_BIND: false
#SERVER_SOCKET_BIND_NODE: ""
#CLIENT_SOCKET_BIND_NODE: ""
#RUN_SINGLE_NODE: false
```
2. Two node test with one of NUMA nodes used:
```
TPCC_NUM_WAREHOUSES: 1000
DB_CPU_LIMIT: 64
#TPCC_HAMMER_NUM_VIRTUAL_USERS: "112_120_128_136_144" # First run VU scaling and find the peak VU
TPCC_HAMMER_NUM_VIRTUAL_USERS: "128" # Second run with fixed VU(peak VU found in VU scaling) for serveral times
ENABLE_SOCKET_BIND: true
SERVER_SOCKET_BIND_NODE: "0" # run MySQL on NUMA node 0 of server node
CLIENT_SOCKET_BIND_NODE: "0" # run HammerDB on NUMA node 0 of client node

# comment out since we are using default value
#RUN_SINGLE_NODE: false
```

3. Single node test with MySQL on NUMA node 0 and HammerDB on NUMA node 1:
```
TPCC_NUM_WAREHOUSES: 1000
DB_CPU_LIMIT: 64
#TPCC_HAMMER_NUM_VIRTUAL_USERS: "112_120_128_136_144" # First run VU scaling and find the peak VU
TPCC_HAMMER_NUM_VIRTUAL_USERS: "128" # Second run with fixed VU(peak VU found in VU scaling) for serveral times
ENABLE_SOCKET_BIND: true
SERVER_SOCKET_BIND_NODE: "0" # run MySQL on NUMA node 0 of node
CLIENT_SOCKET_BIND_NODE: "1" # run HammerDB on NUMA node 1 of node
RUN_SINGLE_NODE: true
```

#### Run the testcase with proper test config
You can run the test case with following command:
```shell
./ctest.sh -R test_static_hammerdb_tpcc_mysql8031_base_disk_hugepage_off_pkm --set TPCC_NUM_WAREHOUSES=2000 --set DB_CPU_LIMIT=128 --set TPCC_HAMMER_NUM_VIRTUAL_USERS="256"
```
Or you can create a test-config.yaml:
```yaml
*:
    TPCC_NUM_WAREHOUSES: 2000
    DB_CPU_LIMIT: 128
    #TPCC_HAMMER_NUM_VIRTUAL_USERS: "224_240_256_272_288" # First run VU scaling and find the peak VU
    TPCC_HAMMER_NUM_VIRTUAL_USERS: "256" # Second run with fixed VU(peak VU found in VU scaling) for serveral times
```
and run with:
```shell
./ctest.sh -R test_static_hammerdb_tpcc_mysql8031_base_disk_hugepage_off_pkm --config=<path-to-test-config-yaml>
```

### KPI
Run the [`list-kpi.sh`](/script/benchmark/list-kpi.sh) script to parse the KPIs from the validation logs. 

The expected output will be similar to this. Please note that the numbers might be slightly different. 
Primary KPI is `Peak New Orders Per Minute (orders/min)` which has a * as prefix

```
New Orders Per Minute xxx (orders/min): xxxxx
Transactions Per Minute xxx (trans/min): xxxxx
Peak Num of Virtual Users: xxx
*Peak New Orders Per Minute (orders/min): xxxxx
Peak Transactions Per Minute (trans/min): xxxxx
```

### Index Info
- Name: `HammerDB-TPCC`  
- Category: `DataServices`  
- Platform: `ICX`  
- Keywords: `MYSQL`  
- Permission:   

### See Also

- [`HammerDB Official Website`](https://www.hammerdb.com/)

