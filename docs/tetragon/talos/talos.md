---
title: Talos How To
layout: default
parent: Tetragon
nav_order: 2
---

```bash
oc project grafana-operator
oc create ns tetragon

# Install Tetragon https://docs.isovalent.com/operations-guide/tetragon/installation/openshift.html
oc -n tetragon edit cm tetragon-operator-config

# Set serviceMonitorEnabled: true
# There are 2 instances of this key 


oc label ns tetragon openshift.io/cluster-monitoring="true"

oc adm policy add-cluster-role-to-user cluster-monitoring-view -z grafana-sa
oc create token grafana-sa --duration=4294967296s
# SAVE THE TOKEN and 

cat << EOF | oc apply -f -
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: prometheus
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana-a"
  datasource:
    name: Prometheus
    u
    type: prometheus
    access: proxy
    url: https://thanos-querier.openshift-monitoring.svc.cluster.local:9091
    isDefault: true
    jsonData:
      timeInterval: "5s"
      httpHeaderName1: 'Authorization'
      tlsSkipVerify: true
    secureJsonData:
      httpHeaderValue1: 'Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6InBVUWk4b216bWJPYjlycUdPdzVNSzhkSlpndTg4STlrRFNybGFTTWZlRkEifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIl0sImV4cCI6NjAzNzc1MDQ1OSwiaWF0IjoxNzQyNzgzMTYzLCJpc3MiOiJodHRwczovL2t1YmVybmV0ZXMuZGVmYXVsdC5zdmMiLCJqdGkiOiIyNDMzNDYyZS1iM2ZlLTQyMTUtOTI1Ni03MjNmNmQyNzZmNTAiLCJrdWJlcm5ldGVzLmlvIjp7Im5hbWVzcGFjZSI6ImdyYWZhbmEtb3BlcmF0b3IiLCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZ3JhZmFuYS1zYSIsInVpZCI6ImRhNjUyNWU3LWUxZTktNGExZC1iZTE0LWQ1MzE3YzhjOTFkMiJ9fSwibmJmIjoxNzQyNzgzMTYzLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6Z3JhZmFuYS1vcGVyYXRvcjpncmFmYW5hLXNhIn0.NaxXVxEKnEmlUUAD33YkJ9hDoQ4Lkl4MD4zAYnfTdHrAI9WtW1uS43kyWzAjHb0AySdGwKqF44f8JqmG5_4qIhBy4GaRAhOJJtjJW8d7Kh70zuJuW5upvYhRlE3LDUljq066OsMHemETQBC067Rv5GcI-v-LOxoXluMnWk2yFA2aOZKifPE_gesxQMNtl8n1B1X4mQ0W6CRjJrElvO4nCbe8KKRJFOJzUthrtRoRp1DEDd92A95kbtbgMYYI0iGi85e40svIUtEcskfIrozdhGQ0bUc1Bxgq00dKYiby0XCI7cN1vmKIcqHOBMATMPWJo6BjzY48889dSEuC8OKg0UpXTIZhQk_dWJJh0YUkz32LjTxdsFmGxIC-aq1C_bWGvrUuoUJsum0_-18bvTr19Ir92VTyEeh5_SzBh74lBx3SXDVJutz6gG2EY_1pnHns8dGxr2Lv9WGsZwLhbA6NftEITxs7KP39cANsyV9YkzrL9tPeSRmdNrLDSwFdjfe2Nr7qX8SKKxiWzv5m6alEKulw9S62w35gx5UKURQ0CtsHgJ8iOau8yOqmlHrrMcyOEfRmB9nPbg7xT_k1MrmrlYXOlYR-4z5WJ5fQylVoWUkbgxhhw0iaVYle6TRBdE-C1BiYng4lMEOqLTG4oNsQokGR8xJj-kZ3YuSoNNAA5ds'
EOF

# Add the role so that the prometheus SA can read the POD/Services 
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