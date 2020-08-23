
**汇总拓扑:**

| host | cpu  | memory  | disk  | deploy |
| --- | --- | --- | --- | --- |
|192.168.5.227  | 4core | 32g  | 209g HP SSD EX950 1TB  | pd + tidb +tikv + alertmanager |
| 192.168.5.228 | 4core |  32g | 184g HP SSD EX950 1TB   | pd + tidb +tikv + prometheus |
| 192.168.5.229 | 4core | 32g  | 265g HP SSD EX950 1TB  | pd + tidb +tikv + grafana |

-------------



**一. 安装tiup:**
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
6. tiup cluster start tidb-test (stop , destroy to fix timezone)
But  it comes up with ' unknown or incorrect time zone: SystemV/PST8PDT' 
8. 重装:
    tiup cluster stop tidb-test
    tiup cluster destroy tidb-test
    tiup cluster deploy tidb-test v4.0.0 ./topology.yaml --user nuc 
    tiup cluster start tidb-test
    修复了上面问题
9. 验证部署情况:
    1. tiup cluster list
    2. tiup cluster display tidb-test
    3. mysql -u root -h 192.168.5.227 -P 4000
    4. 访问dashboard http://192.168.5.228:2379/dashboard/#/overview (密码不用输入)
    


**二. 测试(悲观事务):**
1. **准备grafana 监控:**(初始账号密码admin admin)
    关键指标
    1. **Tidb Summary-Query Summary:** http://192.168.5.229:3000/d/000000012/tidb-test-tidb-summary?orgId=1
    2. **Tikv Details-Cluster CPU:** http://192.168.5.229:3000/d/RDVQiEzZz/tidb-test-tikv-details?orgId=1
    3. **Tikv Details-gRPC ops 和 gRPC duration:** http://192.168.5.229:3000/d/RDVQiEzZz/tidb-test-tikv-details?orgId=1
    
    耗时分析:
    1. **profile:** http://192.168.5.228:2379/dashboard/#/instance_profiling
    
2. **开始压测:** (参考https://github.com/pingcap-incubator/tidb-in-action/blob/master/session4/chapter3/sysbench.md 写配置更快)
 
    1. **https://github.com/akopytov/sysbench**
        1. create database benchmark;
        2. config:
            mysql-host=192.168.5.227
            mysql-port=4000
            mysql-user=root
            mysql-password=
            mysql-db=benchmark
            time=180
            threads=128
            report-interval=10
            db-driver=mysql

        2. **Prepare:** sysbench --config-file=/home/nuc/benchmark/sysbench_thread128.cfg oltp_point_select --tables=16 --table-size=5000000 prepare
        3. **Point Select**: sysbench --config-file=/home/nuc/benchmark/sysbench_thread128.cfg oltp_point_select --tables=16 --table-size=5000000 run
            1. 测试结果:
                ```
                SQL statistics:
                    queries performed:
                        read:                            7323669
                        write:                           0
                        other:                           0
                        total:                           7323669
                    transactions:                        7323669 (40682.84 per sec.)
                    queries:                             7323669 (40682.84 per sec.)
                    ignored errors:                      0      (0.00 per sec.)
                    reconnects:                          0      (0.00 per sec.)

                General statistics:
                    total time:                          180.0175s
                    total number of events:              7323669

                Latency (ms):
                         min:                                    0.11
                         avg:                                    3.15
                         max:                                   48.92
                         95th percentile:                        7.17
                         sum:                             23036623.30

                Threads fairness:
                    events (avg/stddev):           57216.1641/205.11
                    execution time (avg/stddev):   179.9736/0.01
                 ```
                              
            2. ![sysbench_tidb-query-summary](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/sysbench/sysbench_tidb-query-summary.png)
            3. ![sysbench_tikv-details-cpu](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/sysbench/sysbench_tikv-details-cpu.png)
            4. ![sysbench_tikv-details-gRPC](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/sysbench/sysbench_tikv-details-gRPC.png)
        4. **Update Index**: sysbench --config-file=/home/nuc/benchmark/sysbench_thread128.cfg oltp_update_index --tables=16 --table-size=5000000 run
        5.  **Read Only**: sysbench --config-file=/home/nuc/benchmark/sysbench_thread128.cfg oltp_read_only --tables=16 --table-size=5000000 run
    2. **https://github.com/pingcap/go-ycsb**
        1. cd /home/nuc/benchmark/go-ycsb;./bin/go-ycsb  load mysql -P workloads/workloada -p recordcount=5000000 -p mysql.host=192.168.5.227 -p mysql.port=4000 --threads 128
        2. cd /home/nuc/benchmark/go-ycsb;./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=5000000 -p mysql.host=192.168.5.227 -p mysql.port=4000 --threads 128
        3. 测试结果:
            ```
            READ   - Takes(s): 801.8, Count: 2498354, OPS: 3116.1, Avg(us): 8938, Min(us): 389, Max(us): 499807, 99th(us): 35000, 99.9th(us): 53000, 99.99th(us): 184000
            
            UPDATE - Takes(s): 801.7, Count: 2501582, OPS: 3120.2, Avg(us): 31846, Min(us): 1775, Max(us): 1614822, 99th(us): 195000, 99.9th(us): 286000, 99.99th(us): 421000
            ```
            ![go-ycsb_tidb-query-summary](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/go-ycsb/go-yscb_tidb-query-summary.png)      
             ![go-yscb_tikv-details-cpu](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/go-ycsb/go-yscb_tikv-details-cpu.png)
             ![go-yscb_tidb-tikv-details-gRPC](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/go-ycsb/go-yscb_tidb-tikv-details-gRPC.png)
    3. **https://github.com/pingcap/go-tpc**
        1. go-tpc tpcc -H 192.168.5.227 -P 4000 -D tpcc --warehouses 100  prepare -T 16
        2. go-tpc tpcc -H 192.168.5.227 -P 4000 -D tpcc --warehouses 100  run -T 16
        3. go-tpc tpcc -H 192.168.5.227 -P 4000 -D tpcc --warehouses 100  check -T 16
        4. go-tpc tpcc -H 192.168.5.227 -P 4000 -D tpcc --warehouses 100  cleanup -T 16
        5. **测试结果:**
        ```
        DELIVERY - Takes(s): 2569.9, Count: 32500, TPM: 758.8, Sum(ms): 5893715, Avg(ms): 181, 95th(ms): 256, 99th(ms): 512, 99.9th(ms): 512
        
        DELIVERY_ERR - Takes(s): 0.0, Count: 1, TPM: 6355.1, Sum(ms): 19, Avg(ms): 19, 95th(ms): 20, 99th(ms): 20, 99.9th(ms): 20
        
        NEW_ORDER - Takes(s): 2570.2, Count: 363147, TPM: 8477.6, Sum(ms): 20018025, Avg(ms): 55, 95th(ms): 96, 99th(ms): 128, 99.9th(ms): 192
        
        NEW_ORDER_ERR - Takes(s): 0.0, Count: 7, TPM: 43877.5, Sum(ms): 254, Avg(ms): 36, 95th(ms): 64, 99th(ms): 64, 99.9th(ms): 64
        
        ORDER_STATUS - Takes(s): 2570.2, Count: 32262, TPM: 753.1, Sum(ms): 510239, Avg(ms): 15, 95th(ms): 32, 99th(ms): 48, 99.9th(ms): 96
        
        PAYMENT - Takes(s): 2570.2, Count: 346496, TPM: 8088.8, Sum(ms): 13569020, Avg(ms): 39, 95th(ms): 80, 99th(ms): 96, 99.9th(ms): 160
        
        PAYMENT_ERR - Takes(s): 0.0, Count: 3, TPM: 18770.5, Sum(ms): 37, Avg(ms): 12, 95th(ms): 24, 99th(ms): 24, 99.9th(ms): 24
        
STOCK_LEVEL - Takes(s): 2570.2, Count: 32086, TPM: 749.0, Sum(ms): 726191, Avg(ms): 22, 95th(ms): 40, 99th(ms): 64, 99.9th(ms): 80
        ```
        ![go-tpc_tidb-query-summary](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/go-tpc/go-tpc_tidb-query-summary.png)
        ![go-tpc_tikv-details-cpu](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/go-tpc/go-tpc_tikv-details-cpu.png)
        ![go-tpc_tikv-details-gRPC](https://github.com/gdgrc/pingcap_lesson/blob/master/lesson2/go-tpc/go-tpc_tikv-details-gRPC.png)
        

