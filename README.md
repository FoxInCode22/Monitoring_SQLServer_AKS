# Monitoring Microsoft SQL Server in AKS using Prometheus, Grafana and SQL Exporter

As organizations increasingly adopt containerized data platforms on Kubernetes, Microsoft SQL Server deployments on Azure Kubernetes Service (AKS) are becoming more common for development, testing, and even production workloads. While Kubernetes-native monitoring for application workloads is well established, many customers still have questions around the monitoring and observability capabilities available specifically for SQL Server running inside containers. Traditional SQL Server monitoring approaches were primarily designed for virtual machines or bare-metal environments, and there is often uncertainty about how these operational insights translate into Kubernetes-based deployments.  

In this blog, we will implement a Kubernetes-native monitoring solution for SQL Server running in AKS using Prometheus, Grafana, and sql_exporter. The objective is to demonstrate how SQL Server performance and operational metrics can be exposed as Prometheus metrics and visualized through Grafana dashboards alongside existing Kubernetes observability tooling.
The monitoring stack implemented in this guide enables visibility into several important SQL Server operational metrics, including:
-	 SQL Server availability and exporter health
-	 Active and total database connections
-	 Memory utilization
-	 Database size metrics
-	 IO statistics
-	 CPU-related metrics
-	 Blocking sessions and deadlocks
-	 Transaction and workload activity
  
Using Prometheus and ServiceMonitor resources, the exporter integrates directly into the Kubernetes monitoring ecosystem, allowing SQL Server metrics to be scraped and retained alongside cluster-level telemetry. Grafana dashboards can then be used to visualize trends, analyze workload behavior, and create operational dashboards for database monitoring.

By the end of this guide, we will have the following monitoring architecture operational inside AKS:

MSSQL → sql_exporter → Prometheus → Grafana

## SQL Exporter

SQL Exporter is a widely used open-source monitoring tool that enables observability for SQL databases by exposing database metrics in a format consumable by Prometheus. Developed as a database-agnostic exporter, it supports platforms such as SQL Server, PostgreSQL, MySQL, and others, allowing teams to monitor diverse environments using a unified approach. It is highly flexible and configuration-driven, where custom SQL queries can be defined and grouped into collectors to generate meaningful metrics like query performance, connections, or I/O statistics. This makes SQL Exporter a powerful choice for implementing custom, query-level monitoring in modern cloud-native architectures.

## Overview

This guide walks through setting up a production-style monitoring stack for Microsoft SQL Server running in Azure Kubernetes Service (AKS) using:
- Microsoft SQL Server 
- sql_exporter 
- Prometheus 
- Grafana 
- ServiceMonitor 
- kube-prometheus-stack

## Architecture
```
AKS Cluster
│
├── MSSQL Pod
│
├── sql_exporter Pod
│     └── Exposes Prometheus metrics
│
├── Prometheus
│     └── Scrapes sql_exporter metrics
│
└── Grafana
      └── Visualizes metrics

```
## Deployment Order 
Follow this exact sequence:

1. SQL Server Deployment
2. SQL Server Service
3. Prometheus + Grafana stack
4. SQL Exporter ConfigMap
5. SQL Exporter Deployment + Service
6. ServiceMonitor
7. Prometheus validation
8. Grafana dashboard setup

Step 1 : Deploy SQL Server
```
kubectl apply -f sqlserver-deployment.yaml
```
After each deployment of the pod verify if the deployment is successful.
```
kubectl get pods
```
Expected :
```
sqlserver-xxxxx   1/1 Running
```

Step 2 : Create SQL Server Service
```
kubectl apply -f sqlsvc.yaml
```
Verify using,
```
kubectl get svc
```
This service acts as the internal endpoint for SQL Exporter.

Step 3 : Install Prometheus & Grafana

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack
```
Components installed:

- Prometheus
- Grafana
- Alertmanager
- Node exporters

Verify and Wait until all monitoring pods are running.

Step 4 : Configure SQL Exporter
```
kubectl apply -f sql_ex_config.yaml
```

This contains:

SQL connection string
Custom queries (collectors)
Metric mappings

Verify,
```
kubectl get configmap
```

Step 5 : Deploy SQL Exporter
```
kubectl apply -f sql_exporter.yaml
```
Verify again and see if all pods are healthy.

<img width="899" height="245" alt="image" src="https://github.com/user-attachments/assets/15a3d7d8-754c-44eb-8048-25227b3aa422" />


Step 6 : Validate SQL Exporter Metrics
```
kubectl port-forward svc/sql-exporter 9399:9399
```
Open ``` http://localhost:9399/metrics ``` and validate if the expected metrics are seen.

Note : If only go_* metrics appear, configuration is not loading correctly.

<img width="871" height="613" alt="image" src="https://github.com/user-attachments/assets/0aef292e-b869-4cfb-9c8f-312a477e77bd" />

Step 7 : Configure ServiceMonitor
```
kubectl apply -f sql-exporter-servicemonitor.yaml
```

Step 8 : Validate Prometheus Scraping
```
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
```
Test Metrics in Prometheus
```
mssql_up
mssql_connections_total
mssql_memory_used_mb
```
<img width="865" height="424" alt="image" src="https://github.com/user-attachments/assets/5a4ecd14-da86-4fba-b085-d5b0b8f7224d" />

Step 9 : Access Grafana
```
kubectl port-forward svc/monitoring-grafana 3000:80
```
Open: ```http://localhost:3000```

Retieve grafana admin passwrod using
```
kubectl get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | %{[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))}
```

Step 11 : Build Grafana Dashboard
Create dashboard → Prometheus datasource

Sample Queries:
| Metric | Query |
|------|------|
| SQL Availability | `mssql_up` |
| Connections | `mssql_connections_total` |
| Memory Usage | `mssql_memory_used_mb` |
| Database Size | `mssql_database_size_mb` |


<img width="1531" height="945" alt="image" src="https://github.com/user-attachments/assets/597109bf-df8d-41ca-836d-3d4639bf4fa4" />


Note : The approach outlined in this guide is primarily intended as a POC for learning, validation, and initial setup. As part of this, we use port-forwarding to access Prometheus and Grafana locally.
However, port-forwarding is temporary and tied to your local session and not suitable for production environments.
In a Production environment, Grafana and Prometheus should be exposed securely without relying on kubectl port-forward.    
There are various approaches such as,	
- Expose Services via LoadBalancer
- Use Ingress Controller (recommended)

## Summary 
In this guide, we implemented a Kubernetes-native monitoring solution for Microsoft SQL Server running in AKS using Prometheus, Grafana, and SQL Exporter. The setup demonstrates how SQL Server metrics can be integrated into the broader Kubernetes observability ecosystem, enabling centralized monitoring for both infrastructure and database workloads.

While the implementation shown here focuses on a proof-of-concept and learning-oriented setup, the same architecture can be extended further for enterprise-grade deployments by introducing secure ingress, persistent storage, alerting, authentication, and advanced dashboarding capabilities. The flexibility of SQL Exporter and the extensibility of Prometheus make this approach highly adaptable for monitoring modern containerized SQL Server environments.

As more organizations modernize database platforms onto Kubernetes, having visibility into SQL Server health, workload behavior, and resource utilization becomes critical for operational reliability. This monitoring stack provides a strong foundation for building scalable and cloud-native observability solutions for SQL Server workloads running in AKS.

### References :
https://github.com/burningalchemist/sql_exporter

