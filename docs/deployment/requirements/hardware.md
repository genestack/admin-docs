# Hardware

- We recommend allocating 8 cores and 64GB of memory for POC/Trial environments (with small number of datasets) and 16 cores 128GB of memory for production environments.
- It is necessary to provide multiple disks for databases and internal services. You can use any solution that works with Kubernetes (AWS EBS, Azure Disks, etc.). An important note: for optimal performance, we strongly recommend using SSDs.
- The operating system can be any type and any version of Linux (Ubuntu, CentOS, RedHat, Amazon Linux, etc). Preferably, use the latest versions.
