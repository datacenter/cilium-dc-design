---
title: OpenShift How To
layout: default
parent: Tetragon
nav_order: 1
---

{: .warning }
You will most likely have to customize the YAML Examples or CLI/Versions in this section to suit your needs

# Installing Tetragon

In order to Install tetragon there are a few steps we need to follow:

1) We start by creating a namespace for it and labeling it so that it will be scraped by the OpenShift Prometheus instance.

```bash
oc create ns tetragon
oc label ns tetragon openshift.io/cluster-monitoring="true"
```

2) Deployment of Tetragon Catalog

```bash
cat <<EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: tetragon-catalog
  namespace: tetragon
spec:
  sourceType: grpc
  image: quay.io/isovalent/tetragon-operator-index:v0.0.1
EOF
```

{: .warning }
At the time of writing the most recent version of the tetragon-operator-index is `v0.0.1` but you should verify that is still the case by checking the official [Tetragon Installation Guide](https://docs.isovalent.com/operations-guide/tetragon/installation/openshift.html)

3) After a minute or so you should see the `Teragon Operator` in the `OperatorHub` page of OpenShift. Simply click on Install the configuration will be done afterwards.

![alt text](../tetragon-operator.png)

{: .note }
At the time of writing the most recent version of the tetragon is `1.15.0`

Edit the Tetragon Operator config and ensure the `--metrics-bind-address=:2113` is present as show below.

```bash
oc edit ClusterServiceVersion  tetragon-operator.v1.15.0

#Find this section

containers:
- args:
  - serve
  - --config-dir=/etc/tetragon/operator.conf.d/
  - --metrics-bind-address=:2113
```

4) Enable the `serviceMonitor` feature so that Prometheus can scrape the metrics by editing the `tetragon-operator-config` and restart the Tetragon Operator to pick up the config changes
```bash
oc -n tetragon edit cm tetragon-operator-config
## Set to true the 2 instances of the serviceMonitorEnabled variable. Yes there are 2 Instances!

oc delete pod -l app.kubernetes.io/name=tetragon-operator

oc -n tetragon get svc
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
tetragon            ClusterIP   172.31.245.244   <none>        2112/TCP    23h
tetragon-catalog    ClusterIP   172.31.47.22     <none>        50051/TCP   2d4h
tetragon-operator   ClusterIP   172.31.247.192   <none>        2113/TCP    40m
```
5) Create a TracingPolicy to collect the metrics we care about

```bash
cat << EOF | oc apply -n tetragon -f -
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: network-tracing
spec:
  parser:
    dns:
      enable: true
    interface:
      enable: true
      packet: true
    tcp:
      enable: true
      statsInterval: 20
    udp:
      cgroup: true
      enable: true
      statsInterval: 20
    http:
        enable: true
        selectors:
        - matchPorts:
          - 8080
          - 80 
EOF

# Check they are applied correctly
oc exec -n tetragon  daemonsets/tetragon -- tetra tracingpolicy list
ID   NAME              STATE     FILTERID   NAMESPACE   SENSORS                                                                         KERNELMEMORY
1    network-tracing   enabled   0          (global)    __networkPacket_probe__,__parser_sensors__,__sockops_sensors__,layer3_sensors   305.30 kB
```

{: .warning }
Even if Cilium is the only installed CNI the OpenShift node might still be loading the openvswitch kernel module. If this happens the Tracing policy will not be enabled successfully and you might see this error message in all the Tetragon pods:

```shell
oc logs -n tetragon daemonsets/tetragon | grep network-tracing    
Found 8 pods, using pod/tetragon-wlhnq
time="2025-03-24T03:35:40Z" level=warning msg="adding tracing policy failed" error="sensor __networkPacket_probe__ from collection network-tracing failed to load: failed prog /var/lib/tetragon/bpf_dev_queue_xmit.o kern_version 331264 loadInstance: opening collection '/var/lib/tetragon/bpf_dev_queue_xmit.o' failed: populating kallsyms caches: getting modules from kallsyms: assigning symbol modules: symbol dev_queue_xmit: duplicate found at address 0xffffffffc105e020 (module \"openvswitch\"): multiple kernel symbols with the same name" 
```

You can check if the module is loaded by executing this command:

```
oc debug node/<node_name>  --  /host/usr/sbin/lsmod | grep openvswitch
openvswitch           245760  0
nf_conncount           24576  1 openvswitch 
```

If this is the case you can follow this procedure to blacklist the module so that is not loaded:
[How to blacklist a kernel module in OpenShift 4.x](https://access.redhat.com/solutions/6979679)


6) Create a role in the tetragon namespace so that Prometheus can scrape it:

```bash
cat << EOF | oc apply -n tetragon -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prometheus-k8s
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prometheus-k8s
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: openshift-monitoring
EOF
```

In the `Observe` --> `Targets` you should now see `tetragon` as an entry:

![alt text](../prometheus-targets.png)



# Grafana

OpenShift comes pre-installed with its own Dashboards under `Observe` --> `Dashboards` but they are not using Grafana and as such we need to install our own instance.
This is a pretty easy tasks:

1) Install the `Grafana Operator` from the `OperatorHub`
   
2) Create a `Grafana` Instance below an example for an instance that uses PVC to save dashboards and is exposed via a `route` 

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: Grafana
metadata:
  name: grafana
  labels:
    dashboards: grafana
    folders: grafana
  namespace: grafana-operator
spec:
  config:
    auth:
      disable_login_form: 'false'
    log:
      mode: console
    security:
      admin_password: start
      admin_user: root
  deployment:
    spec:
      template:
        spec:
          volumes:
            - persistentVolumeClaim:
                claimName: grafana-pvc
              name: grafana-data
  route:
    spec:
      host: grafana.apps.ocp-cilium-c1.cam.ciscolabs.com
  persistentVolumeClaim:
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi
      storageClassName: lvms-vg1
```

3) We need now to tell Grafana to use the in-cluster prometheus instance as a `DataSource` to do this we need to:
  - Add the `grafana ServiceAccount` to the `cluster-monitoring-view` Role
  - Create a `token` to use to Authenticate towards Prometheus
  - Create a `GrafanaDatasource` 

```bash
oc adm policy add-cluster-role-to-user cluster-monitoring-view -z grafana-sa
TOKEN=`oc create token grafana-sa --duration=4294967296s`

cat << EOF | oc apply -f -
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: prometheus
  namespace: grafana-operator
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  datasource:
    name: prometheus
    type: prometheus
    access: proxy
    url: https://thanos-querier.openshift-monitoring.svc:9091
    isDefault: true
    jsonData:
      timeInterval: "5s"
      httpHeaderName1: 'Authorization'
      tlsSkipVerify: true
    secureJsonData:
      httpHeaderValue1: 'Bearer ${TOKEN}'
EOF
```

4) The Grafana Operator uses the `GrafanaDashboard` CRD to add new dashboards. For simplicity we have collected a few dashboards [here](https://github.com/datacenter/cilium-dc-design/tree/main/docs/tetragon/grafana/operator/dashboards) you should be able to just apply them as they are. Start by applying `01-folders.yaml` first.