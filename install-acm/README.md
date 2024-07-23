Note the following can be accomplished via the GUI console as well. This is just the CLI method.  
  
Log into the primary OpenShift cluster

1. create a project for ACM
   `oc new-project open-cluster-management`
3. create the OperatorGroup object
   `oc create -f operator-group.yaml`
4. create the Subscription object
   `oc create -f subscription.yaml`
5. patch install plan
```
$ oc get installplan
NAME            CSV                                  APPROVAL   APPROVED
install-4k2q8   advanced-cluster-management.v2.4.2   Manual     false

oc patch installplan install-4k2q8 \
  --type merge --patch '{"spec":{"approved":true}}'
installplan.operators.coreos.com/install-4k2q8 patched
```
6. Watch the status of the available ClusterServiceVersion objects to verify the installation status. Wait until the PHASE column shows the Succeeded message.
```
$ watch oc get csv
...output omitted...
NAME                                 DISPLAY                                      VERSION ... PHASE
advanced-cluster-management.v2.4.2   Advanced Cluster Management for Kubernetes   2.4.2   ... Succeeded
```
7. Create the RHACM MultiClusterHub object
   `oc create -f mch.yaml`
8. Wait until the MultiClusterHub object creates all its components. Use the watch command to monitor the status.
```
$ watch oc get multiclusterhub
...output omitted...
NAME              STATUS       AGE
multiclusterhub   Installing   3m21s
```

9. Prepare RHACM to import a cluster using the name managed-cluster
```
oc new-project managed-cluster

oc label namespace managed-cluster \
  cluster.open-cluster-management.io/managedCluster=managed-cluster
```

10. Create the ManagedCluster object
    `oc create -f mngcluster.yaml`

11. Create a klusterlet add-on configuration
    `oc create -f klusterlet.yaml`

12. Obtain the necessary files to import the managed-cluster cluster.

```
oc get secret managed-cluster-import \
  -n managed-cluster -o jsonpath={.data.crds\\.yaml} | base64 \
  --decode > klusterlet-crd.yaml

oc get secret managed-cluster-import \
  -n managed-cluster -o jsonpath={.data.import\\.yaml} | base64 \
  --decode > import.yaml
```

13. Log into the second cluster that is to be managed by ACM
14. Use the klusterlet-crd.yaml file to create the klusterlet custom resource definition.
    `oc create -f klusterlet-crd.yaml`
15. use the import.yaml file to create the rest of the resources necessary to import the cluster into RHACM
    `oc create -f import.yaml`
16. Verify the status of the pods running in the open-cluster-management-agent namespace.
```
$ oc get pod -n open-cluster-management-agent
NAME                                             READY   STATUS    RESTARTS   AGE
klusterlet-bfb4cd68f-j2fdt                       1/1     Running   0          53s
klusterlet-registration-agent-5f749f5cf9-8xvp6   1/1     Running   0          36s
klusterlet-registration-agent-5f749f5cf9-pnkj6   1/1     Running   0          36s
klusterlet-registration-agent-5f749f5cf9-slk8j   1/1     Running   0          36s
klusterlet-work-agent-7d848f496b-7mnfw           1/1     Running   0          36s
klusterlet-work-agent-7d848f496b-bwkfv           1/1     Running   1          36s
klusterlet-work-agent-7d848f496b-wn2vd           1/1     Running   0          36s
```
17. use the watch command to validate the status of the agent pods running in the open-cluster-management-agent-addon namespace.
```
$ watch oc get pod -n open-cluster-management-agent-addon
NAME                                                         READY   STATUS    RESTARTS   AGE
klusterlet-addon-appmgr-7f69b84c76-kvhp7                     1/1     Running   0          55s
klusterlet-addon-certpolicyctrl-7c97656db8-5s6xk             1/1     Running   0          54s
klusterlet-addon-iampolicyctrl-6f8cccf86c-lj2fg              1/1     Running   0          54s
klusterlet-addon-operator-b479bb446-kpc6t                    1/1     Running   0          83s
klusterlet-addon-policyctrl-config-policy-769745c7b6-78sw2   1/1     Running   0          54s
klusterlet-addon-policyctrl-framework-6455bc9558-bz8mc       3/3     Running   0          54s
klusterlet-addon-search-7b5bb78f98-h2jlg                     1/1     Running   0          53s
klusterlet-addon-workmgr-5b84cfd6dc-nxwvt
```
18. Log back into the hub cluster and verify the managed-cluster status.
19. verify managed-cluster status
```
$ oc get managedcluster
NAME              HUB ACCEPTED   ...   JOINED   AVAILABLE   AGE
local-cluster     true           ...   True     True        4h26m
managed-cluster   true           ...   True     True        42m
```
