## Master Nodes
- lucky-km1 
	- 10.25.18.31
- lucky-km2
	- 10.25.18.32
- lucky-km3
	- 10.25.18.33

## Worker Nodes
- lucky-kw1
	- 10.25.18.34
- lucky-kw2
	- 10.25.18.35
- lucky-kw3
	- 10.25.18.36

## High-Level Architecture Flow
```
Users / Admins / External Clients
        │
        ▼
Public Domains
        │
        ├── api.pem.gg
        ├── harbor.pem.gg
        ├── lucky.pem.gg
        ├── admin.pem.gg
        ├── tma.pem.gg
        ├── grafana.pem.gg
        ├── minio.pem.gg
        ├── s3.minio.pem.gg
        └── k8.dashboard.pem.gg
        │
        ▼
DNS Resolution
        │
        ▼
Public IP: 103.71.40.14
        │
        ▼
Firewall / NAT / Load Balancer
        │
        ▼
Kubernetes Worker Node IPs
        │
        ├── lucky-kw1: 10.25.18.34
        ├── lucky-kw2: 10.25.18.35
        └── lucky-kw3: 10.25.18.36
        │
        ▼
Nginx Ingress Controller
        │
        ├── HTTP  NodePort: 30548
        └── HTTPS NodePort: 30225
        │
        ▼
Kubernetes Ingress Rules
        │
        ▼
Kubernetes Services
        │
        ▼
Application / Platform Pods
```

## Physical / Virtualization Layer
```
Physical Server / Infrastructure
        │
        ▼
Proxmox Node: pve2-gpu
        │
        ├── Master VM: lucky-km1
        │   ├── Role: Kubernetes Control Plane
        │   ├── IP: 10.25.18.31
        │   ├── CPU: 4 cores
        │   ├── RAM: 4 GB
        │   └── Boot Disk: 50 GB
        │
        ├── Master VM: lucky-km2
        │   ├── Role: Kubernetes Control Plane
        │   ├── IP: 10.25.18.32
        │   ├── CPU: 4 cores
        │   ├── RAM: 4 GB
        │   └── Boot Disk: 50 GB
        │
        ├── Master VM: lucky-km3
        │   ├── Role: Kubernetes Control Plane
        │   ├── IP: 10.25.18.33
        │   ├── CPU: 4 cores
        │   ├── RAM: 4 GB
        │   └── Boot Disk: 50 GB
        │
        ├── Worker VM: lucky-kw1
        │   ├── Role: Kubernetes Worker
        │   ├── IP: 10.25.18.34
        │   ├── CPU: 16 cores
        │   ├── RAM: 16 GB
        │   └── Boot Disk: 200 GB
        │
        ├── Worker VM: lucky-kw2
        │   ├── Role: Kubernetes Worker
        │   ├── IP: 10.25.18.35
        │   ├── CPU: 16 cores
        │   ├── RAM: 16 GB
        │   └── Boot Disk: 200 GB
        │
        └── Worker VM: lucky-kw3
            ├── Role: Kubernetes Worker
            ├── IP: 10.25.18.36
            ├── CPU: 16 cores
            ├── RAM: 16 GB
            └── Boot Disk: 200 GB
```

## Kubernetes Cluster Layer
```
Kubernetes Cluster
        │
        ├── Control Plane
        │   ├── lucky-km1: 10.25.18.31
        │   ├── lucky-km2: 10.25.18.32
        │   └── lucky-km3: 10.25.18.33
        │
        └── Worker Nodes
            ├── lucky-kw1: 10.25.18.34
            ├── lucky-kw2: 10.25.18.35
            └── lucky-kw3: 10.25.18.36
```

## Control Plane Architecture 
```
Control Plane Nodes
        │
        ├── lucky-km1
        │   ├── kube-apiserver
        │   ├── kube-controller-manager
        │   ├── kube-scheduler
        │   ├── etcd
        │   ├── kube-proxy
        │   └── calico-node
        │
        ├── lucky-km2
        │   ├── kube-apiserver
        │   ├── kube-controller-manager
        │   ├── kube-scheduler
        │   ├── etcd
        │   ├── kube-proxy
        │   └── calico-node
        │
        └── lucky-km3
            ├── kube-apiserver
            ├── kube-controller-manager
            ├── kube-scheduler
            ├── etcd
            ├── kube-proxy
            └── calico-node
```

## Worker Node Architecture 
```
Worker Layer
        │
        ├── lucky-kw1: 10.25.18.34
        │   ├── ingress-nginx-controller
        │   ├── cert-manager
        │   ├── Kubernetes Dashboard
        │   ├── Harbor core/database/redis/jobservice
        │   ├── MinIO pod
        │   ├── Redis cache / queue pods
        │   ├── MySQL cluster pod/router
        │   ├── Alertmanager
        │   ├── JMeter / InfluxDB
        │   └── luckypem-tma
        │
        ├── lucky-kw2: 10.25.18.35
        │   ├── ingress-nginx-controller
        │   ├── Grafana
        │   ├── kube-state-metrics
        │   ├── MySQL cluster pod/router
        │   ├── Redis cache / queue pods
        │   ├── MinIO pod
        │   ├── phpMyAdmin
        │   ├── mysql-operator
        │   ├── luckypem-admin
        │   └── umami
        │
        └── lucky-kw3: 10.25.18.36
            ├── ingress-nginx-controller
            ├── Prometheus
            ├── MinIO pod
            ├── MySQL cluster pod/router
            ├── Harbor registry/portal/trivy
            ├── Redis cache / queue pods
            ├── luckypem-backend
            ├── luckypem-frontend
            └── discord-bot
```

## Data and Platform Service Layer
```
Data / Platform Services
        │
        ├── MySQL Cluster
        │   ├── mycluster-0
        │   ├── mycluster-1
        │   ├── mycluster-2
        │   └── mycluster-router x3
        │
        ├── Redis Cache
        │   ├── redis-cache-0
        │   ├── redis-cache-1
        │   ├── redis-cache-2
        │   ├── redis-cache-sentinel-0/1/2
        │   └── redis-cache-haproxy x2
        │
        ├── Redis Queue
        │   ├── redis-queue-0
        │   ├── redis-queue-1
        │   ├── redis-queue-2
        │   ├── redis-queue-sentinel-0/1/2
        │   └── redis-queue-haproxy x2
        │
        ├── MinIO Object Storage
        │   ├── minio-0
        │   ├── minio-1
        │   └── minio-2
        │
        ├── Harbor Registry
        │   ├── harbor-core
        │   ├── harbor-registry
        │   ├── harbor-database
        │   ├── harbor-redis
        │   ├── harbor-jobservice
        │   ├── harbor-portal
        │   └── harbor-trivy
        │
        └── Monitoring
            ├── Prometheus
            ├── Grafana
            ├── Alertmanager
            ├── kube-state-metrics
            └── node-exporter
```

## Storage Architecture
```
Persistent Storage  
│  
▼  
StorageClass: local-path  
│  
├── MySQL PVCs  
│ ├── datadir-mycluster-0: 50 Gi  
│ ├── datadir-mycluster-1: 50 Gi  
│ └── datadir-mycluster-2: 50 Gi  
│  
├── MinIO PVCs  
│ ├── data-minio-0: 20 Gi  
│ ├── data-minio-1: 20 Gi  
│ └── data-minio-2: 20 Gi  
│  
├── Harbor PVCs  
│ ├── harbor-registry: 20 Gi  
│ ├── harbor-database: 5 Gi  
│ ├── harbor-trivy: 5 Gi  
│ ├── harbor-redis: 2 Gi  
│ └── harbor-jobservice: 2 Gi  
│  
├── Redis PVCs  
│ ├── redis-cache: 5 Gi each  
│ ├── redis-queue: 10 Gi each  
│ └── redis-sentinel: 1 Gi each  
│  
└── Monitoring PVCs  
├── Prometheus: 30 Gi  
├── Grafana: 5 Gi  
└── Alertmanager: 2 Gi
```
