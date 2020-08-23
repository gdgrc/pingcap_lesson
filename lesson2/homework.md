
192.168.5.227  4core 32g 636/886 209g 76%  good
192.168.5.228      658/886 184 79%   good
192.168.5.229 8c  32g  576/886 265 69%  good

-------------

**192.168.5.227:**

**安装tiup:**
1. curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
2. source ~/.profile
3. tiup cluster
4. 3台机器互相免密码登陆，登陆用户拥有免密码sudo权限
sudo chmod +w /etc/sudoers
sudo bash
sudo echo "nuc    ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
5. 准备topology.yaml
```
# # Global variables are applied to all deployments and used as the default value of
# # the deployments if a specific deployment value is missing.
global:
  user: "nuc"
  ssh_port: 22
  deploy_dir: "/home/nuc/bin/tidb-deploy"
  data_dir: "/home/nuc/data/tidb-data"
  log_dir: "/home/nuc/log/tidb-log"

# # Monitored variables are applied to all the machines.
monitored:
  node_exporter_port: 9100
  blackbox_exporter_port: 9115
  # deploy_dir: "/tidb-deploy/monitored-9100"
  # data_dir: "/tidb-data/monitored-9100"
  # log_dir: "/tidb-deploy/monitored-9100/log"

# # Server configs are used to specify the runtime configuration of TiDB components.
# # All configuration items can be found in TiDB docs:
# # - TiDB: https://pingcap.com/docs/stable/reference/configuration/tidb-server/configuration-file/
# # - TiKV: https://pingcap.com/docs/stable/reference/configuration/tikv-server/configuration-file/
# # - PD: https://pingcap.com/docs/stable/reference/configuration/pd-server/configuration-file/
# # All configuration items use points to represent the hierarchy, e.g:
# #   readpool.storage.use-unified-pool
# #      
# # You can overwrite this configuration via the instance-level `config` field.

server_configs:
  tidb:
    log.slow-threshold: 300
    binlog.enable: false
    binlog.ignore-error: false
  tikv:
    # server.grpc-concurrency: 4
    # raftstore.apply-pool-size: 2
    # raftstore.store-pool-size: 2
    # rocksdb.max-sub-compactions: 1
    # storage.block-cache.capacity: "16GB"
    # readpool.unified.max-thread-count: 12
    readpool.storage.use-unified-pool: false
    readpool.coprocessor.use-unified-pool: true
  pd:
    schedule.leader-schedule-limit: 4
    schedule.region-schedule-limit: 2048
    schedule.replica-schedule-limit: 64

pd_servers:
  - host: 192.168.5.227
    # ssh_port: 22
    # name: "pd-1"
    # client_port: 2379
    # peer_port: 2380
    # deploy_dir: "/tidb-deploy/pd-2379"
    # data_dir: "/tidb-data/pd-2379"
    # log_dir: "/tidb-deploy/pd-2379/log"
    # numa_node: "0,1"
    # # The following configs are used to overwrite the `server_configs.pd` values.
    # config:
    #   schedule.max-merge-region-size: 20
    #   schedule.max-merge-region-keys: 200000
  - host: 192.168.5.228
  - host: 192.168.5.229

tidb_servers:
  - host: 192.168.5.227
    # ssh_port: 22
    # port: 4000
    # status_port: 10080
    # deploy_dir: "/tidb-deploy/tidb-4000"
    # log_dir: "/tidb-deploy/tidb-4000/log"
    # numa_node: "0,1"
    # # The following configs are used to overwrite the `server_configs.tidb` values.
    # config:
    #   log.slow-query-file: tidb-slow-overwrited.log
  - host: 192.168.5.228
  - host: 192.168.5.229

tikv_servers:
  - host: 192.168.5.227
    # ssh_port: 22
    # port: 20160
    # status_port: 20180
    # deploy_dir: "/tidb-deploy/tikv-20160"
    # data_dir: "/tidb-data/tikv-20160"
    # log_dir: "/tidb-deploy/tikv-20160/log"
    # numa_node: "0,1"
    # # The following configs are used to overwrite the `server_configs.tikv` values.
    # config:
    #   server.grpc-concurrency: 4
    #   server.labels: { zone: "zone1", dc: "dc1", host: "host1" }
  - host: 192.168.5.228
  - host: 192.168.5.229

monitoring_servers:
  - host: 192.168.5.228
    # ssh_port: 22
    # port: 9090
    # deploy_dir: "/tidb-deploy/prometheus-8249"
    # data_dir: "/tidb-data/prometheus-8249"
    # log_dir: "/tidb-deploy/prometheus-8249/log"

grafana_servers:
  - host: 192.168.5.229
    # port: 3000
    # deploy_dir: /tidb-deploy/grafana-3000

alertmanager_servers:
  - host: 192.168.5.227
    # ssh_port: 22
    # web_port: 9093
    # cluster_port: 9094
    # deploy_dir: "/tidb-deploy/alertmanager-9093"
    # data_dir: "/tidb-data/alertmanager-9093"
    # log_dir: "/tidb-deploy/alertmanager-9093/log"

```
5. tiup cluster deploy tidb-test v4.0.0 ./topology.yaml --user nuc
```
Please confirm your topology:
tidb Cluster: tidb-test
tidb Version: v4.0.0
Type          Host           Ports        OS/Arch       Directories
----          ----           -----        -------       -----------
pd            192.168.5.227  2379/2380    linux/x86_64  /home/nuc/bin/tidb-deploy/pd-2379,/home/nuc/data/tidb-data/pd-2379
pd            192.168.5.228  2379/2380    linux/x86_64  /home/nuc/bin/tidb-deploy/pd-2379,/home/nuc/data/tidb-data/pd-2379
pd            192.168.5.229  2379/2380    linux/x86_64  /home/nuc/bin/tidb-deploy/pd-2379,/home/nuc/data/tidb-data/pd-2379
tikv          192.168.5.227  20160/20180  linux/x86_64  /home/nuc/bin/tidb-deploy/tikv-20160,/home/nuc/data/tidb-data/tikv-20160
tikv          192.168.5.228  20160/20180  linux/x86_64  /home/nuc/bin/tidb-deploy/tikv-20160,/home/nuc/data/tidb-data/tikv-20160
tikv          192.168.5.229  20160/20180  linux/x86_64  /home/nuc/bin/tidb-deploy/tikv-20160,/home/nuc/data/tidb-data/tikv-20160
tidb          192.168.5.227  4000/10080   linux/x86_64  /home/nuc/bin/tidb-deploy/tidb-4000
tidb          192.168.5.228  4000/10080   linux/x86_64  /home/nuc/bin/tidb-deploy/tidb-4000
tidb          192.168.5.229  4000/10080   linux/x86_64  /home/nuc/bin/tidb-deploy/tidb-4000
prometheus    192.168.5.228  9090         linux/x86_64  /home/nuc/bin/tidb-deploy/prometheus-9090,/home/nuc/data/tidb-data/prometheus-9090
grafana       192.168.5.229  3000         linux/x86_64  /home/nuc/bin/tidb-deploy/grafana-3000
alertmanager  192.168.5.227  9093/9094    linux/x86_64  /home/nuc/bin/tidb-deploy/alertmanager-9093,/home/nuc/data/tidb-data/alertmanager-9093

```
6. tiup cluster start tidb-test (stop , destroy to fix timezone)
But  it comes up with ' unknown or incorrect time zone: SystemV/PST8PDT' 
8. 重装:
    tiup cluster stop tidb-test
    tiup cluster destroy tidb-test
    tiup cluster deploy tidb-test v4.0.0 ./topology.yaml --user nuc 
    tiup cluster start tidb-test
    修复了上面问题
9. tiup cluster list
```
Starting component `cluster`: /home/nuc/.tiup/components/cluster/v1.0.9/tiup-cluster list
Name       User  Version  Path                                                PrivateKey
----       ----  -------  ----                                                ----------
tidb-test  nuc   v4.0.0   /home/nuc/.tiup/storage/cluster/clusters/tidb-test  /home/nuc/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa
```
10. tiup cluster display tidb-test
```
Starting component `cluster`: /home/nuc/.tiup/components/cluster/v1.0.9/tiup-cluster display tidb-test
tidb Cluster: tidb-test
tidb Version: v4.0.0
ID                   Role          Host           Ports        OS/Arch       Status  Data Dir                                    Deploy Dir
--                   ----          ----           -----        -------       ------  --------                                    ----------
192.168.5.227:9093   alertmanager  192.168.5.227  9093/9094    linux/x86_64  Up      /home/nuc/data/tidb-data/alertmanager-9093  /home/nuc/bin/tidb-deploy/alertmanager-9093
192.168.5.229:3000   grafana       192.168.5.229  3000         linux/x86_64  Up      -                                           /home/nuc/bin/tidb-deploy/grafana-3000
192.168.5.227:2379   pd            192.168.5.227  2379/2380    linux/x86_64  Up      /home/nuc/data/tidb-data/pd-2379            /home/nuc/bin/tidb-deploy/pd-2379
192.168.5.228:2379   pd            192.168.5.228  2379/2380    linux/x86_64  Up|UI   /home/nuc/data/tidb-data/pd-2379            /home/nuc/bin/tidb-deploy/pd-2379
192.168.5.229:2379   pd            192.168.5.229  2379/2380    linux/x86_64  Up|L    /home/nuc/data/tidb-data/pd-2379            /home/nuc/bin/tidb-deploy/pd-2379
192.168.5.228:9090   prometheus    192.168.5.228  9090         linux/x86_64  Up      /home/nuc/data/tidb-data/prometheus-9090    /home/nuc/bin/tidb-deploy/prometheus-9090
192.168.5.227:4000   tidb          192.168.5.227  4000/10080   linux/x86_64  Up      -                                           /home/nuc/bin/tidb-deploy/tidb-4000
192.168.5.228:4000   tidb          192.168.5.228  4000/10080   linux/x86_64  Up      -                                           /home/nuc/bin/tidb-deploy/tidb-4000
192.168.5.229:4000   tidb          192.168.5.229  4000/10080   linux/x86_64  Up      -                                           /home/nuc/bin/tidb-deploy/tidb-4000
192.168.5.227:20160  tikv          192.168.5.227  20160/20180  linux/x86_64  Up      /home/nuc/data/tidb-data/tikv-20160         /home/nuc/bin/tidb-deploy/tikv-20160
192.168.5.228:20160  tikv          192.168.5.228  20160/20180  linux/x86_64  Up      /home/nuc/data/tidb-data/tikv-20160         /home/nuc/bin/tidb-deploy/tikv-20160
192.168.5.229:20160  tikv          192.168.5.229  20160/20180  linux/x86_64  Up      /home/nuc/data/tidb-data/tikv-20160         /home/nuc/bin/tidb-deploy/tikv-20160
```
11. 验证
    1. mysql -u root -h 192.168.5.227 -P 4000
    2. 访问dashboard http://192.168.5.228:2379/dashboard/#/overview (密码不用输入)
    
 12. 测试前准备
     tiup cluster edit-config tidb-test:
     1. tidb日志级别error.    log.level: error
     2. 开启 TiDB 配置中的 prepared plan cache，可减少优化执行计划的开销: prepared_plan_cache.enabled: true(这个加不上)
     3. 本地事务冲突检测设置，并发压测时建议开启，可减少事务的冲突txn_local_latches.enabled: true

     2. tiup cluster reload tidb-test
     
 (参数总览)
 ```
    server_configs:
  tidb:
    binlog.enable: false
    binlog.ignore-error: false
    log.level: error
    log.slow-threshold: 300
  tikv:
    global.log-level: error
    raftstore.sync-log: false
    readpool.coprocessor.use-unified-pool: true
    readpool.storage.use-unified-pool: false
    storage.block-cache.capacity: 10GB
  pd:
    schedule.leader-schedule-limit: 4
    schedule.region-schedule-limit: 2048
    schedule.replica-schedule-limit: 64
```

13. 测试. 悲观事务
    1. https://github.com/akopytov/sysbench (参考https://github.com/pingcap-incubator/tidb-in-action/blob/master/session4/chapter3/sysbench.md 写配置更快)
        1. create database benchmark;
        2. sysbench --config-file=sysbench_thread64.cfg oltp_point_select --tables=16 --table-size=5000000 prepare
        3. **Point Select**: sysbench --config-file=sysbench_thread64.cfg oltp_point_select --tables=16 --table-size=5000000 run
        4. **Update Index**: sysbench --config-file=sysbench_thread64.cfg oltp_update_index --tables=16 --table-size=5000000 run
        5.  **Read Only**: sysbench --config-file=sysbench_thread64.cfg oltp_read_only --tables=16 --table-size=5000000 run
    3. https://github.com/pingcap/go-ycsb
        1. cd /home/nuc/benchmark/go-ycsb;./bin/go-ycsb  load mysql -P workloads/workloada -p recordcount=5000000 -p mysql.host=192.168.5.227 -p mysql.port=4000 --threads 64
        2. cd /home/nuc/benchmark/go-ycsb;./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=5000000 -p mysql.host=192.168.5.227 -p mysql.port=4000 --threads 64
    2. https://github.com/pingcap/go-tpc
        1. go-tpc tpcc -H 192.168.5.227 -P 4000 -D tpcc --warehouses 100  prepare -T 8
        2. go-tpc tpcc -H 192.168.5.227 -P 4000 -D tpcc --warehouses 100  run -T 8
        3. go-tpc tpcc -H 192.168.5.227 -P 4000 -D tpcc --warehouses 100  check -T 8
        4. go-tpc tpcc -H 192.168.5.227 -P 4000 -D tpcc --warehouses 100  cleanup -T 8

13. grafana 监控，账号密码admin admin
