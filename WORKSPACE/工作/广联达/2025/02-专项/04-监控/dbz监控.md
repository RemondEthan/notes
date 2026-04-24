~~~json
1、当前正常在快照的表名：修正增加快照状态
2、快照剩余条数：通过TableFrom和TableTo来体现
3、记录handleBatch的binlog位点：EmbeddedEngineChangeEvent类是非public，拿不到sourceRecord，没有gtid信息，且只记录了最后一个非心跳Event的信息

{
    "streamingMetrics": {
        "GtidSet": "61a14907-303c-11ee-917e-32c6ffb7ee8f:1-23283015",
        "MilliSecondsSinceLastEvent": 0,
        "MilliSecondsBehindSource": 90707094,
        "BinlogFilename": "mysql-bin.000652",
        "BinlogPosition": 194
    },
    "mysqlSourceConsumerMetrics": {
        "sinkTableUpdateTimeMetrics": {
            "987654336_test_liurx.test00001": "2025-02-08 17:29:55"
        },
        "lastHandleEvent": {
            "file": "mysql-bin.000652",
            "pos": "66471848",
            "name": "987654336",
            "gtids": null,
            "version": "gdmp-2.3.0.Final-0.11"
        },
        "tableStatusMetrics": {
            "test_liurx.valuation_fbfx_lmm_detail": "存量数据准备中",
            "test_liurx.test00005": "存量数据准备中",
            "test_liurx.test00001_target": "存量数据准备中",
            "test_liurx.test00001": "存量数据同步中"
        }
    },
    "snapshotMetrics": {
        "TableTo": null,
        "ChunkFrom": null,
        "RemainingTableCount": 0,
        "CapturedTables": [],
        "ChunkId": "04f73206-d7cd-4744-8afa-6d7338beafc1",
        "ChunkTo": null,
        "SnapshotRunning": true,
        "RowsScanned": {
            "test_liurx.test00001": 4378
        },
        "TableFrom": null
    }
}

{
    "streamingMetrics": {
        "GtidSet": "61a14907-303c-11ee-917e-32c6ffb7ee8f:1-23283139",
        "MilliSecondsSinceLastEvent": 4000,
        "MilliSecondsBehindSource": 178827,
        "BinlogFilename": "mysql-bin.000652",
        "BinlogPosition": 66551346
    },
    "mysqlSourceConsumerMetrics": {
        "sinkTableUpdateTimeMetrics": {
            "987654336_test_liurx.valuation_fbfx_lmm_detail": "2025-02-08 17:30:15",
            "987654336_test_liurx.test00005": "2025-02-08 17:30:11",
            "987654336_test_liurx.test00001": "2025-02-08 17:30:08"
        },
        "lastHandleEvent": {
            "file": "mysql-bin.000652",
            "pos": "66551346",
            "name": "987654336",
            "gtids": null,
            "version": "gdmp-2.3.0.Final-0.11"
        },
        "tableStatusMetrics": {
            "test_liurx.valuation_fbfx_lmm_detail": "存量数据同步中",
            "test_liurx.test00005": "增量数据同步中",
            "test_liurx.test00001_target": "增量数据同步中",
            "test_liurx.test00001": "增量数据同步中"
        }
    },
    "snapshotMetrics": {
        "TableTo": "[212157813429731347]",
        "ChunkFrom": "[200252123775074304]",
        "RemainingTableCount": 0,
        "CapturedTables": [],
        "ChunkId": "9895dc59-a491-410c-a3b0-c0c34fed643b",
        "ChunkTo": "[210800748469968950]",
        "SnapshotRunning": true,
        "RowsScanned": {
            "test_liurx.valuation_fbfx_lmm_detail": 20000,
            "test_liurx.test00005": 2,
            "test_liurx.test00001": 4378
        },
        "TableFrom": "[200252123775074304]"
    }
}

{
    "streamingMetrics": {
        "GtidSet": "61a14907-303c-11ee-917e-32c6ffb7ee8f:1-23283167",
        "MilliSecondsSinceLastEvent": 0,
        "MilliSecondsBehindSource": 177932,
        "BinlogFilename": "mysql-bin.000652",
        "BinlogPosition": 66566535
    },
    "mysqlSourceConsumerMetrics": {
        "sinkTableUpdateTimeMetrics": {
            "987654336_test_liurx.valuation_fbfx_lmm_detail": "2025-02-08 17:30:41"
        },
        "lastHandleEvent": {
            "file": "mysql-bin.000652",
            "pos": "66560053",
            "name": "987654336",
            "gtids": null,
            "version": "gdmp-2.3.0.Final-0.11"
        },
        "tableStatusMetrics": {
            "test_liurx.valuation_fbfx_lmm_detail": "增量数据同步中",
            "test_liurx.test00005": "增量数据同步中",
            "test_liurx.test00001_target": "增量数据同步中",
            "test_liurx.test00001": "增量数据同步中"
        }
    },
    "snapshotMetrics": {
        "TableTo": null,
        "ChunkFrom": null,
        "RemainingTableCount": 0,
        "CapturedTables": [],
        "ChunkId": null,
        "ChunkTo": null,
        "SnapshotRunning": false,
        "RowsScanned": {
            "test_liurx.valuation_fbfx_lmm_detail": 47580,
            "test_liurx.test00005": 2,
            "test_liurx.test00001": 4378
        },
        "TableFrom": null
    }
}

{
    "streamingMetrics": {
        "GtidSet": "61a14907-303c-11ee-917e-32c6ffb7ee8f:1-23282129",
        "MilliSecondsSinceLastEvent": 0,
        "MilliSecondsBehindSource": 179659,
        "BinlogFilename": "mysql-bin.000652",
        "BinlogPosition": 65918876
    },
    "mysqlSourceConsumerMetrics": {
        "sinkTableUpdateTimeMetrics": {
            "987654336_test_liurx.valuation_fbfx_lmm_detail": "2025-02-08 17:09:12",
            "987654336_test_liurx.test00005": "2025-02-08 17:09:12",
            "987654336_test_liurx.test00001": "2025-02-08 17:09:10"
        },
        "lastHandleEvent": {
            "file": "mysql-bin.000652",
            "pos": "65918876",
            "name": "987654336",
            "version": "gdmp-2.3.0.Final-0.11"
        },
        "tableStatusMetrics": {
            "test_liurx.valuation_fbfx_lmm_detail": "存量数据准备中",
            "test_liurx.test00005": "增量数据同步中",
            "test_liurx.test00001_target": "增量数据同步中",
            "test_liurx.test00001": "增量数据同步中"
        }
    },
    "snapshotMetrics": {
        "TableTo": "[212157813429731347]",
        "ChunkFrom": "[200252123775074304]",
        "RemainingTableCount": 2,
        "CapturedTables": [
            "test_liurx.valuation_fbfx_lmm_detail",
            "test_liurx.test00001_target",
            "test_liurx.test00005",
            "test_liurx.test00001"
        ],
        "ChunkId": "96c074c9-b4d4-4f64-8e68-caae6f711836",
        "ChunkTo": "[210800748469968950]",
        "SnapshotRunning": true,
        "RowsScanned": {
            "test_liurx.valuation_fbfx_lmm_detail": 20000,
            "test_liurx.test00005": 2,
            "test_liurx.test00001": 10000
        },
        "TableFrom": "[200252123775074304]"
    }
}
~~~
