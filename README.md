# Workload Onboarding on Managed Clusters through OpenShift GitOps
## Deploying Applications to Managed Clusters using GitOps Operator

## TL;DR
Lveraging ACM (Application Configuration Management) and GitOps Operator for deploying OCP (OpenShift) clusters and applications onto them.
When the spoke cluster is deployed with ZTP, ArgoCD only see local cluster as target cluster. In order to add the managed cluster as target cluster and create argo applications, you need to create ManagedClusterSet along with some bidning rules. The purpose of this post is to share the my experiences and provide insights into this process.

## Precondition
For the purpose of this example, it is assumed that the spoke cluster "sno1" has already been deployed and imported into the ACM. This step is necessary to ensure that the placement rule can be triggered as part of the subsequent process. Additionally, it is assumed that sno1 has been carried out using the siteconfig generator.

## Addon-application-Manager
First, it is important to ensure that the managed cluster has the "feature.open-cluster-management.io/addon-application-manager": "available" label applied. ,br>
This label indicates that the Application Manager feature is available and enabled on the cluster. 

To verify the label, you can use the following command:
```
$ oc get managedcluster <cluster-name> -n open-cluster-management-agent-addon -o jsonpath='{.metadata.labels}'
```

## ManagedClusterSet and binding rules
To create a ManagedClusterSet custom resource (CR), you can use the following example YAML definition:

```
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: myworkload
```

### ManagedClusterSetBinding
```
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSetBinding
metadata:
  name: myworkload
  namespace: openshift-gitops
spec:
  clusterSet: myworkload
```

### Placement
```
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: all-openshift-clusters
  namespace: openshift-gitops
spec:
  tolerations:
  - key: cluster.open-cluster-management.io/unreachable
    operator: Exists
  - key: cluster.open-cluster-management.io/unavailable
    operator: Exists
  predicates:
  - requiredClusterSelector:
      labelSelector:
        matchExpressions:
        - key: sites
          operator: In
          values:
          - sno1
```
### GitOpsCluster

```
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: gitops-cluster-sample
  namespace: openshift-gitops
spec:
  argoServer:
    argoNamespace: openshift-gitops
    cluster: local-cluster
  placementRef:
    apiVersion: cluster.open-cluster-management.io/v1beta1
    kind: Placement
    name: all-openshift-clusters
```

## Move spoke cluster to the cluster set
To include the  "sno1" cluster in the cluster set using the oc label command, execute the following command:
```
$ oc label Managedcluster sno1 cluster.open-cluster-management.io/clusterset=myworkload --overwrite
```
Now you should see the spoke cluster in the cluster list in the ArogCD.

![image](https://github.com/mabehiro/app-onboard-acmgitops/assets/104660935/15a9a196-e5e5-4560-84a8-77f1b1c38e5c)
