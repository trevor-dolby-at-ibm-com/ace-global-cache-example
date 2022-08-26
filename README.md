# ace-global-cache-example

This repo contains an example using the embedded global cache in ACE and describes how to change the locking
pattern for the global cache maps. Optimistic locking provides much better performance without sacrificing
data coherence for the example flows, and this change is safe due to the auto-commit transactional model used
in the global cache interactions from ACE.

The demo application uses two flows:
- WriteMapDataOnStartup populates the cache on first startup, and updates the version number of each entry on subequent startups. See [WriteMapDataOnStartup_JavaCompute.java](BasicTimingJava/example/globalcache/WriteMapDataOnStartup_JavaCompute.java) for the Java code that implements the flow logic.
- ReadMapData runs every five seconds and reads all of the entries written by WriteMapDataOnStartup ten times. See [ReadMapData_JavaCompute.java](BasicTimingJava/example/globalcache/ReadMapData_JavaCompute.java) for the Java code that implements the flow logic.

Cloning this repo and deploying it locally should allow anyone using ACE v11 or later to recreate the 
results, though ACE v12 is recommended due to having the eGit plugin (and therefore the git perspective) 
included in the toolkit by default. The server to which the flows are deploy must have the global cache 
support turned on and be connected to a global cache grid.

The results are shown in detail below, but in a simple configuration involving two VMs the optimistic 
locking configuration was an order of magnitude faster; see [pessimistic](#pessimistic-locking) and
[optimistic](#optimistic-locking) sections for details on the performance numbers in the simple case. The
[transactional considerations](#transactional-considerations) section provides more details on how
ACE flows interact with global cache grids from a transactional perspective, and explains some of the 
flow results in greater detail.

## Deployment configuration

The results below were obtained using two independent ACE servers, with one running the flows and the other dedicated to 
the global cache itself.

<img src="https://github.com/trevor-dolby-at-ibm-com/ace-global-cache-example/blob/main/two-server.png" width="600"/>

It is also possible to run the entire setup in a single server, though this is less realistic (most 
global cache grids are spread across multiple servers) and shows less of an improvement with locking
changes:

<img src="https://github.com/trevor-dolby-at-ibm-com/ace-global-cache-example/blob/main/single-server.png" width="600"/>

## Pessimistic locking

This is the ACE default behaviour, and so no additional configuration is required.

Running the flows with an empty cache:
```
[lots of log spam on startup]
2022-08-25 18:46:40.338644: BIP1991I: Integration server has finished initialization. 
2022-08-25 18:46:42.590184: BIP7155I: The integration server has established a connection to the embedded global cache. 
WriteMapDataOnStartup setting 1000 entries
2022-08-25 18:46:42.787     40 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:46:50.554     38 WriteMapDataOnStartup finished; numberOfEntriesCreated=1000 numberOfEntriesUpdated=0
2022-08-25 18:46:57.457     40 ReadMapData finished; foundAnyNullValues=true foundDifferentVersions=false
2022-08-25 18:46:57.459     40 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:46:57.461     40 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:47:05.370     40 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:47:05.372     40 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:47:05.374     40 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:47:13.223     40 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:47:13.225     40 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:47:13.227     40 ReadMapData reading entries (10 times 1000 entries)
```
where 10000 reads take around 7500-7800ms once the writer flow has finished. This is faster when run using
a local cache, but still takes several seconds due to the locking requirements.

## Optimistic locking

### Setup

Enabling optimistic locking is described in the ACE documentation (see [references](#references) for links)
but the summary is as follows.

server.conf.yaml snippet needed to configure the flow server assuming the grid is running on 192.168.1.252:
```
---
ResourceManagers:
  GlobalCache:
    cacheOn: true
    catalogServiceEndPoints: '192.168.1.252:2800'
    catalogClusterEndPoints: 'ExampleCatalogServer1:192.168.1.252:2803:2801'
    objectGridCustomFile: '/home/newuser/tmp/gc/objectgrid.xml'
    deploymentPolicyCustomFile: '/home/newuser/tmp/gc/deployment.xml'
```

The map configuration in /home/newuser/tmp/gc/objectgrid.xml contains
```
<?xml version="1.0" encoding="UTF-8"?>
<objectGridConfig xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://ibm.com/ws/objectgrid/config ../objectGrid.xsd"
  xmlns="http://ibm.com/ws/objectgrid/config">
 <objectGrids>
  <objectGrid name="WMB">
   <backingMap name="EXAMPLE.*" template="true" timeToLive="0" ttlEvictorType="LAST_UPDATE_TIME" lockStrategy="OPTIMISTIC_NO_VERSIONING" nearCacheInvalidationEnabled="true" copyMode="COPY_TO_BYTES"/>
   <backingMap name="SYSTEM.BROKER.*" template="true" timeToLive="0" ttlEvictorType="LAST_UPDATE_TIME" lockStrategy="PESSIMISTIC" copyMode="COPY_TO_BYTES"/>
  </objectGrid>
 </objectGrids>
</objectGridConfig>
```
with `lockStrategy="OPTIMISTIC_NO_VERSIONING"` being the critical attribute. See the WXS 
documentation (link in the [references](#references) section) for a full description of the
other tags, including `nearCacheInvalidationEnabled`.

As well as the map configuration itself, /home/newuser/tmp/gc/deployment.xml contains the
map reference for `EXAMPLE.*`:
```
<?xml version="1.0" encoding="UTF-8"?>
<deploymentPolicy xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
  xsi:schemaLocation="http://ibm.com/ws/objectgrid/deploymentPolicy ../deploymentPolicy.xsd" 
  xmlns="http://ibm.com/ws/objectgrid/deploymentPolicy"> 
 <objectgridDeployment objectgridName="WMB">
  <mapSet name="mapSet" numberOfPartitions="13" minSyncReplicas="0" maxSyncReplicas="1" replicaReadEnabled="true" >
   <map ref="EXAMPLE.*"/>
   <map ref="SYSTEM.BROKER.*"/>
   <zoneMetadata>
    <shardMapping shard="P" zoneRuleRef="wmbRule"/>
    <shardMapping shard="S" zoneRuleRef="wmbRule"/>
    <zoneRule name="wmbRule" exclusivePlacement="false">
     <zone name="WMBZone" />
    </zoneRule>
   </zoneMetadata>
  </mapSet>
 </objectgridDeployment>
</deploymentPolicy>
```

Note that these experiments require the XML configuration file changes to be made
on both the flow server and the cache server; only the flow server server.conf.yaml
settings are shown, but the cache server also needs the `objectGridCustomFile` and
`deploymentPolicyCustomFile` settings in order for the options to fully take effect.

### Runing with optimistic locking

Running with an empty cache:
```
[lots of log spam on startup]
2022-08-25 18:35:12.021191: BIP1991I: Integration server has finished initialization. 
2022-08-25 18:35:14.173083: BIP7155I: The integration server has established a connection to the embedded global cache. 
ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:35:14.430     42 WriteMapDataOnStartup setting 1000 entries
2022-08-25 18:35:23.682     42 WriteMapDataOnStartup finished; numberOfEntriesCreated=1000 numberOfEntriesUpdated=0
2022-08-25 18:35:24.843     40 ReadMapData finished; foundAnyNullValues=true foundDifferentVersions=false
2022-08-25 18:35:24.847     40 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:35:24.849     40 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:35:25.365     40 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:35:25.367     40 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:35:25.368     40 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:35:25.675     40 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:35:25.678     40 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:35:25.679     40 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:35:26.006     40 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:35:30.517     40 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:35:30.519     40 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:35:30.880     40 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:35:35.890     40 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:35:35.893     40 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:35:36.267     40 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
```
where 10000 reads take around 350-400ms

With a preloaded cache:
```
[lots of log spam on startup]
2022-08-25 18:39:20.933628: BIP1991I: Integration server has finished initialization. 
2022-08-25 18:39:23.115342: BIP7155I: The integration server has established a connection to the embedded global cache. 
WriteMapDataOnStartup setting 1000 entries
2022-08-25 18:39:23.417     42 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:39:29.490     41 WriteMapDataOnStartup finished; numberOfEntriesCreated=0 numberOfEntriesUpdated=1000
2022-08-25 18:39:31.754     42 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=true
2022-08-25 18:39:31.762     42 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:39:31.765     42 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:39:32.626     42 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:39:32.629     42 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:39:32.632     42 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:39:33.416     42 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:39:34.421     42 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:39:34.425     42 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:39:34.763     42 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:39:39.772     42 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:39:39.774     42 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:39:40.179     42 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
2022-08-25 18:39:44.687     42 ReadMapData looking for EXAMPLE.BASIC.MAP
2022-08-25 18:39:44.690     42 ReadMapData reading entries (10 times 1000 entries)
2022-08-25 18:39:45.104     42 ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=false
```
which shows similar times but slightly different startup messages.

## Transactional considerations

### Overview

Global cache updates using map.put() and other methods are not part of the flow transaction, 
and are instead automatically committed during the execution of the method:

![Transaction flow](transaction-boundary.png)

See the "Transactions and Session methods" section at https://www.ibm.com/docs/en/wxs/8.6.1?topic=applications-using-sessions-access-data-in-grid 
for more information on auto-commit.

This means that any flow transaction rollbacks do not remove values added to the map, and so
other mechanisms must be used to ensure the entries are removed (assuming this is needed). The 
ACE MbGlobalMapSessionPolicy allows JavaCompute nodes to set a time-to-live value (link in the
[references](#references) section) that allows data to automatically expire, and this approach
can prevent the cache from continuously expanding when flow transactions roll back.

### Effects on results shown above

The wrtiter flow creates data map entries with an initial version of 0, and then increments the
value on subsequent restarts. This means that the reader flow can tell if the writer flow is 
running at the same time, as the version numbers will change for some entries while the reader
is reading the map. The reader can also tell if any of the values are null, which indicates the 
writer flow is still populating the cache. All of this is possible because the writer additions
and updates are committed individually, and so the changes become visible immediately rather 
than being committed when the flow commits.

Looking at the results above, the reader reports the following for an empty cache
```
ReadMapData finished; foundAnyNullValues=true foundDifferentVersions=false
```
showing that it read null values for some entries due to the writer not having completed writing.

When restarting the reader and writer flow server, the results are
```
ReadMapData finished; foundAnyNullValues=false foundDifferentVersions=true
```
showing that all the entries were present (no null values), but the version value changed
during the reading of the entries. Each individual map entry will only ever have one version,
but because the writer is committing updates individually, the version of one map entry may
be different from another entry. It is also possible that the reader is reading stale version
number data (where the new value has not propagated to all of the replicas) because the
`replicaReadEnabled` configuration value in deployment.xml is set to `true`, though in this
case there is only one container server and so this should not occur.

## References

ACE global cache locking configuration: https://www.ibm.com/docs/en/app-connect/12.0?topic=data-configuring-locking-strategies

ACE global cache time-to-live: https://www.ibm.com/docs/en/app-connect/12.0?topic=agcbujn-specifying-how-long-data-remains-in-global-cache

Locking strategies from the WXS docs: https://www.ibm.com/docs/en/wxs/8.6.1?topic=overview-locking-strategies

