git clone https://github.com/RachnaDodia/gitops_acm.git

set labels for cluster

prod: env=prod
uat: env=uat

create demo cluster set

oc apply -f ManagedClusterSetBinding.yml

oc apply -f Placement.yml

oc apply -f GitOpsCluster.yml

oc get gitopscluster demo-gitops-cluster -n openshift-gitops     -o=jsonpath='{.status.message}'

ACM GUI:

Applications>Create App>ApplicationsSet

ApplicationSet name:  sample-spring-boot
Argo server:    openshift-gitops
Requeue time: 180 
Git repoURL: https://github.com/RachnaDodia/openshift-cluster-config.git
Revision: master, path: apps/simple
Destination Remote namespace : demo
sync policy: Automatically create namespace if it does not exist
Existing placement: demo-gitops-placement

Check App topology

Login argocd to view apps

delete service on uat cluster

check if synced automatically