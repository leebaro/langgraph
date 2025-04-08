# Cloud SaaS

To deploy a [LangGraph Server](../concepts/langgraph_server.md), follow the how-to guide for [how to deploy to Cloud SaaS](../cloud/deployment/cloud.md).

## Overview

The Cloud SaaS deployment option is a fully managed model for deployment where we manage the [control plane](./langgraph_control_plane.md) and [data plane](./langgraph_data_plane.md) in our cloud.

|                   | [Control Plane](../concepts/langgraph_control_plane.md) | [Data Plane](../concepts/langgraph_data_plane.md) |
|-------------------|-------------------|------------|
| **What is it?** | <ul><li>Control Plane UI for creating deployments and revisions</li><li>Control Plane APIs for creating deployments and revisions</li></ul> | <ul><li>Data plane "listener" for reconciling deployments with control plane state</li><li>LangGraph Servers</li><li>Postgres, Redis, etc</li></ul> |
| **Where is it hosted?** | LangChain's cloud | LangChain's cloud |
| **Who provisions and manages it?** | LangChain | LangChain |

## Architecture

<<<<<<< HEAD
!!! warning "Subject to Change"
    The Cloud SaaS deployment architecture may change in the future.

A high-level diagram of a Cloud SaaS deployment.

![diagram](img/langgraph_cloud_architecture.png)

## Whitelisting IP Addresses

All traffic from `LangGraph Platform` deployments created after January 6th 2025 will come through a NAT gateway.
This NAT gateway will have several static ip addresses depending on the region you are deploying in. Refer to the table below for the list of IP addresses to whitelist:

| US             | EU             |
|----------------|----------------|
| 35.197.29.146  | 34.13.192.67   |
| 34.145.102.123 | 34.147.105.64  |
| 34.169.45.153  | 34.90.22.166   |
| 34.82.222.17   | 34.147.36.213  |
| 35.227.171.135 | 34.32.137.113  | 
| 34.169.88.30   | 34.91.238.184  |
| 34.19.93.202   | 35.204.101.241 |
| 34.19.34.50    | 35.204.48.32   |

## Related

- [Deployment Options](./deployment_options.md)
=======
![Cloud SaaS](./img/self_hosted_control_plane_architecture.png)
>>>>>>> main
