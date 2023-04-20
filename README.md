# PTP Operator
## Hack GPSD

1) connect gpsd output to namedpiped shared between two containers. From gpsd daemon in ptp-operator daemonset:

```
sh-5.1# oc exec -it linuxptp-daemon-rk49w -c gpsd -- sh
sh-5.1# mkfifo /gpsd/data

sh-5.1# gpspipe -r > /gpsd/data &
[1] 2020984
sh-5.1# cat /gpsd/data 
{"class":"VERSION","release":"3.25.1~dev","rev":"release-3.25-26-g8ab2872d3","proto_major":3,"proto_minor":15}
{"class":"DEVICES","devices":[{"class":"DEVICE","path":"/dev/gnss0","driver":"u-blox","subtype":"SW EXT CORE 1.00 (3fda8e),HW 00190000","subtype1":"ROM BASE 0x118B2060,FWVER=TIM 2.20,PROTVER=29.20,MOD=ZED-F9T,GPS;GLO;GAL;BDS,SBAS;QZSS,NAVIC","activated":"2023-04-20T12:30:20.787Z","flags":1,"mincycle":0.02}]}
{"class":"WATCH","enable":true,"json":false,"nmea":true,"raw":0,"scaled":false,"timing":false,"split24":false,"pps":false}
$GNRMC,150638.00,A,4233.01364,N,07112.87794,W,0.005,,200423,,,A,V*0B
$GNGGA,150638.00,4233.01364,N,07112.87794,W,1,09,0.91,56.5,M,-33.0,M,,*45
$GNRMC,150639.00,A,4233.01364,N,07112.87795,W,0.005,,200423,,,A,V*0B
```
2) Data from gpsd can be read from linuxptp-daemon-container :

```
[jnunez@flexran-ru ~]$  oc exec -it linuxptp-daemon-rk49w -c linuxptp-daemon-container -- sh
sh-4.4# cat /gpsd/data 
{"class":"VERSION","release":"3.25.1~dev","rev":"release-3.25-26-g8ab2872d3","proto_major":3,"proto_minor":15}
{"class":"DEVICES","devices":[{"class":"DEVICE","path":"/dev/gnss0","driver":"u-blox","subtype":"SW EXT CORE 1.00 (3fda8e),HW 00190000","subtype1":"ROM BASE 0x118B2060,FWVER=TIM 2.20,PROTVER=29.20,MOD=ZED-F9T,GPS;GLO;GAL;BDS,SBAS;QZSS,NAVIC","activated":"2023-04-20T12:32:14.440Z","flags":1,"mincycle":0.02}]}
{"class":"WATCH","enable":true,"json":false,"nmea":true,"raw":0,"scaled":false,"timing":false,"split24":false,"pps":false}
$GNRMC,150843.00,A,4233.01370,N,07112.87813,W,0.001,,200423,,,A,V*08
$GNGGA,150843.00,4233.01370,N,07112.87813,W,1,09,0.92,57.1,M,-33.0,M,,*44
$GNRMC,150844.00,A,4233.01370,N,07112.87813,W,0.002,,200423,,,A,V*0C
```

## Table of Contents

- [PTP Operator](#ptp-operator)
- [PtpOperatorConfig](#ptpoperatorconfig)
- [PtpConfig](#ptpconfig)
- [Quick Start](#quick-start)

## PTP Operator
Ptp Operator, runs in `openshift-ptp` namespace, manages cluster wide PTP configuration. It offers `PtpOperatorConfig` and `PtpConfig` CRDs and creates `linuxptp daemon` to apply node specific PTP config.

## PtpOperatorConfig
Upon deployment of PTP Operator, it automatically creates a `default` custom resource of `PtpOperatorConfig` kind which contains a configurable option `daemonNodeSelector`, it is used to specify which nodes `linuxptp daemon` shall be created on. The `daemonNodeSelector` will be applied to `linuxptp daemon` DaemonSet `nodeSelector` field and trigger relaunching of `linuxptp daemon`. Ptp Operator only recognizes `default` `PtpOperatorConfig`, use `oc edit PtpOperatorConfig default -n openshift-ptp` to update the `daemonNodeSelector`.

```
$ oc get ptpoperatorconfigs.ptp.openshift.io default -n openshift-ptp -o yaml

apiVersion: v1
items:
- apiVersion: ptp.openshift.io/v1
  kind: PtpOperatorConfig
  metadata:
    creationTimestamp: "2019-10-28T06:53:09Z"
    generation: 3
    name: default
    namespace: openshift-ptp
    resourceVersion: "2356742"
    selfLink: /apis/ptp.openshift.io/v1/namespaces/openshift-ptp/ptpoperatorconfigs/default
    uid: d7286542-34bd-4c79-8533-d01e2b25953e
  spec:
    daemonNodeSelector: {}
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
### Enable PTP events via fast event framework 
PTP Operator supports fast event publisher for events such as PTP state change, os clock out of sync, clock class change and port failure.
Event publisher is enabled by deploying PTP operator with [cloud events framework](https://github.com/redhat-cne/cloud-event-proxy) (based on O-RAN API specifications).
The events are published via HTTP or AMQP transport and available for local subscribers.

#### Enabling fast events
```
$ oc edit ptpoperatorconfigs.ptp.openshift.io default -n openshift-ptp

apiVersion: v1
items:
- apiVersion: ptp.openshift.io/v1
  kind: PtpOperatorConfig
  metadata:
    creationTimestamp: "2019-10-28T06:53:09Z"
    generation: 4
    name: default
    namespace: openshift-ptp
    resourceVersion: "2364095"
    selfLink: /apis/ptp.openshift.io/v1/namespaces/openshift-ptp/ptpoperatorconfigs/default
    uid: d7286542-34bd-4c79-8533-d01e2b25953e
  spec:
    ptpEventConfig:
      enableEventPublisher: true
      # For AMQP transport, transportHost: "amqp://amq-router.amq-router.svc.cluster.local"
      transportHost: "http://ptp-event-publisher-service-NODE_NAME.openshift-ptp.svc.cluster.local:9043"
      storageType: local-sc
    daemonNodeSelector:
      node-role.kubernetes.io/worker: ""
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
## PtpConfig

`PtpConfig` CRD is used to define linuxptp configurations and to which node these 
linuxptp configurations shall be applied. 
The Spec of CR has two major sections. 
The first section `profile` contains `interface`, `ptp4lOpts`, `phc2sysOpts` and `ptp4lConf` options,
the second `recommend` defines profile selection logic.
```
 PTP operator supports T-BC and Ordinary clock which can be configured via ptpConfig
```
### ptpConfig to set up ordinary clock using single interface
``` 
NOTE: following ptp4l/phc2sys opts required when events are enabled 
    ptp4lOpts: "-2 -s --summary_interval -4" 
    phc2sysOpts: "-a -r -m -n 24 -N 8 -R 16"
```
```
apiVersion: ptp.openshift.io/v1
kind: PtpConfig
metadata:
  name: ordinary-clock-ptpconfig
  namespace: openshift-ptp
spec:
  profile:
  - name: "profile1"
    interface: "enp134s0f0"
    ptp4lOpts: "-s -2"
    phc2sysOpts: "-a -r"
  recommend:
  - profile: "profile1"
    priority: 4
    match:
    - nodeLabel: "node-role.kubernetes.io/worker"
```
### ptpConfig to set up boundary clock using multiple interface
``` 
NOTE: following ptp4l/phc2sys opts required when events are enabled 
    ptp4lOpts: "-2 --summary_interval -4" 
    phc2sysOpts: "-a -r -m -n 24 -N 8 -R 16"
```
```
apiVersion: ptp.openshift.io/v1
kind: PtpConfig
metadata:
  name: boundary-clock-ptpconfig
  namespace: openshift-ptp
spec:
  profile:
  - name: "profile1"
    ptp4lOpts: "-s -2"
    phc2sysOpts: "-a -r"
    ptp4lConf: |
      [ens7f0]
      masterOnly 0
      [ens7f1]
      masterOnly 1
      [ens7f2]
      masterOnly 1
  recommend:
  - profile: "profile1"
    priority: 4
    match:
    - nodeLabel: "node-role.kubernetes.io/worker"
```
#### ptpConfig to override offset threshold when events are enabled
```
apiVersion: ptp.openshift.io/v1
kind: PtpConfig
metadata:
  name: event-support-ptpconfig
  namespace: openshift-ptp
spec:
  profile:
  - name: "profile1"
    ...
    ...
    ......   
    ptpClockThreshold:
      holdOverTimeout: 24 # in secs
      maxOffsetThreshold: 100 #in nano secs
      minOffsetThreshold: 100 #in nano secs
  recommend:
  - profile: "profile1"
    priority: 4
    match:
    - nodeLabel: "node-role.kubernetes.io/worker"
    
```
#### ptpConfig to filter 'master offset' and 'delay   filtered' logs
```
apiVersion: ptp.openshift.io/v1
kind: PtpConfig
metadata:
  name: suppress-logs-ptpconfig
  namespace: openshift-ptp
spec:
  profile:
  - name: "profile1"
    ...
    ...
    ......   
    ptpSettings:
      stdoutFilter: "^.*delay   filtered.*$"
      logReduce: "true"
  recommend:
  - profile: "profile1"
    priority: 4
    match:
    - nodeLabel: "node-role.kubernetes.io/worker"
    
```
#### ptpConfig to configure as WPC NIC as GM
```
apiVersion: ptp.openshift.io/v1
kind: PtpConfig
metadata:
  name: ptpconfig-gm
  namespace: openshift-ptp
spec:
  profile:
  - name: "profile1"
    ...
    ...
    ......   
    plugins:
      e810:
        enableDefaultConfig: true
    ts2phcOpts: " "
    ts2phcConf: |
      [nmea]
      ts2phc.master 1
      [global]
      use_syslog  0
      verbose 1
      logging_level 7
      ts2phc.pulsewidth 100000000
      #GNSS module s /dev/ttyGNSS* -al use _0
      ts2phc.nmea_serialport  /dev/ttyGNSS_1700_0
      leapfile  /usr/share/zoneinfo/leap-seconds.list
      [ens2f0]
      ts2phc.extts_polarity rising
      ts2phc.extts_correction 0
  recommend:
  - profile: "profile1"
    priority: 4
    match:
    - nodeLabel: "node-role.kubernetes.io/worker"
    
```

In above examples, `profile1` will be applied by `linuxptp-daemon` to nodes labeled with `node-role.kubernetes.io/worker`.

`xxx-ptpconfig` CR is created with `PtpConfig` kind. `spec.profile` defines profile named `profile1` which contains `interface (enp134s0f0)` to run ptp4l process on, `ptp4lOpts (-s -2)` sysconfig options to run ptp4l process with and `phc2sysOpts (-a -r)` to run phc2sys process with. `spec.recommend` defines `priority` (lower numbers mean higher priority, 0 is the highest priority) and `match` rules of profile `profile1`. `priority` is useful when there are multiple `PtpConfig` CRs defined, linuxptp daemon applies `match` rules against node labels and names from high priority to low priority in order. If any of `nodeLabel` or `nodeName` on a specific node matches with the node label or name where daemon runs, it applies profile on that node.

## Quick Start

To install PTP Operator:
```
$ make deploy
```

To un-install:
```
$ make undeploy
```

