# Kubernetes

- Any distribution of Kubernetes could be used, e.g.: EKS, AKS, Rancher, Vanilla Kubernetes. Of the known limitations, we have not tested with Redhat OpenShift.
- ODM requires to store a state inside of Kubernetes, this is under [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) and the corresponding controller responsability.
- For publishing ODM we use Ingress, it requires a [IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/) and the corresponding controller.
- (Recommendation) To work with [SSL/TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls), for Ingress, provide certificates via Kubernetes secrets or configure them on an external Load Balancer.
