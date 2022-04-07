# Default values

```yaml
mcm:
  enabled: true
  config:
    namespace: ibm-cp4mcm
    blockStorageClass: ocs-storagecluster-ceph-rbd
    fileGidStorageClass: ocs-storagecluster-cephfs
    monitoringEnabled: true
    infrastructureManagementEnabled: true
    entitledRegistrySecret: ibm-management-pull-secret
#    mcmCoreDisabled - During IBM Cloud Pak for Multicloud Management installation, the default services operators are
#    installed by default. If you installed Red Hat® Advanced Cluster Management for Kubernetes in your cluster,
#    set mcmCoreDisabled: true. This setting disables the default ibm-management-mcm operator installation.
#    i.e., RHACM is setup already in the cluster
    rhacmEnabled: true
```