# Kubernetes

- You can use almost any solution, for example: EKS, AKS, On-Prem Kubernetes. Of the known limitations, we have not tested with Redhat OpenShift.
- To work with disks, you will need a [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) and the corresponding controller.
- To work with Ingress, you will need a [IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/) and the corresponding controller.
- To work with [SSL/TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls), you need to provide certificates for the Ingress or configure them on an external Load Balancer.
