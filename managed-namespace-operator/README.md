## managed-namespace-operator

This is an ansible operator the is designed to create namespaces with quotas and limits based on the creation of ManagedNamespace custom resources. 

## Install

1) Create project for operator
```bash
oc new-project managed-namespace-operator
```
2) Create the manifests
```bash
oc create -f deploy/crds/better_v1alpha1_managednamespace_crd.yaml
oc create -f deploy/cluster_role.yaml
oc create -f deploy/cluster_role_binding.yaml
oc create -f deploy/service_account.yaml
```
3) Deploy the operator

Note that you need to update the image to wherever you are hosting the image
```bash
oc create -f deploy/operator.yaml
```

## Clean up
```bash
oc delete project managed-namespace-operator
oc delete clusterrole managed-namespace-operator
oc delete clusterrolebinding managed-namespace-operator
```

