# Hardware

- We recommend allocating 16 cores 128GB of memory for production environments.
- It is necessary to provide multiple disks for databases and internal services. You can use any solution that works with Kubernetes (AWS EBS, Azure Disks, etc.). An important note: for optimal performance, we strongly recommend using SSDs.
- The operating system can be any distribution of Linux (Ubuntu, CentOS, RedHat, Amazon Linux, etc). Preferably, use the latest versions.
- The container runtime processor architecture must be [x86-64](https://en.wikipedia.org/wiki/X86-64)
