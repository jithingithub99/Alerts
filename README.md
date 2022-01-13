### Alerts ###

#### 1) KubePersistentVolumeUsageCritical #####
#### 2) KubePersistentVolumeFullInFourDays #####


These alerts are due to  low disk space issue in the pvc.We will have to increase the PVC size

First get the pvc details.

` kubectl get pvc pvcname `

` kubectl describe pvc pvcname `

Verify which storage class it belongs to.Confirm "AllowVolumeExpansion" is set to true

` kubectl describe sc storageclassname `

check whether this pod is controlled by any replicaset or deployment

` kubectl describe pod podname `   (refer the line controlled by)

` kubectl describe deployment deploymentname ` (run if the pod is controlled by deployment)

scale down the replica of deployment to 0

` kubectl scale deployment deploymentname --replicas=0 -n namespace `

Now edit the pvc configuration file

` kubectl edit pvc pvcname `

go to below section in the file


  ``` 
  resources:
  
      requests:
      
           storage: 30Gi > increase to whatever required size
```
           

Now after some time we can see PVC size got increased

verify with  ` kubectl get pvc pvcname `

Now scale up the deployment replica to previous value

` kubectl scale deployment deploymentname --replicas=0 -n namespace `

verify pod is running fine

#### 1) KubePodOOMKilled #####

Sometimes if users run extensive workload at their cluster, it could happen that the default resource limits are not enough. So for this it's needed to potential override the default resource limits of the user cluster.


`kubectl get pods -A | grep -iv running`


Describe the pod to get more information:

`kubectl describe pod -n namespace podname`

we will see some error like OOMKilled and this pod may controlled by statefulset

OOMKilled indicates that assigned memory limit  is not efficient enough. If you take a look at the StatefulSet object you also see that resource limits are maybe set to low:

`kubectl get sts stsname -o yaml | kexp`

Note: kexp is a command of fubectl to extract the temporary status values of the object.


As the Kubermatic cluster controller is managing the StatefulSet object, it's not possible to override the resource limits directly in the StatefulSet object. it would be possible you set the spec.pause=true, but this breaks the update and control behaviour of KKP, as pause means, to disable the cluster controller at all.

A better and constant solution is to overwrite the value at the spec.componentsOverride field, similar as described for e.g. etcd in the official KKP documentation - Scaling the Control Plane.(If the pod is controlled by deployment we can directly edit deployment and update the resource limit & request values-make sure we scale down the replica first)

` kubectl edit cluster clustername`


 ``` apiVersion: kubermatic.k8s.io/v1
     kind: Cluster
     metadata:`
     name: xxxxx  
     spec:
  componentsOverride:
      ####################### <<<<<<<< update
    prometheus: 
      resources:
        limits:
          cpu: 300m
          memory: 3Gi > ### increase memory size ###
        requests:
          cpu: 150m
          memory: 750Mi
  ####################### <<<<<<<< update
  ```
  
  
After the edit, the cluster reconciliation should automatically patch the StatefulSet and trigger a rolling deployment of the prometheus pods:

`kubectl get sts stsname -o yaml | kexp`

Check again if the pod is crashing:

`kubectl get pods -n namespace podname | grep -iv running`

we could see that pod is still in ### CrashLoopBackOff ###

As the StatefulSet contains the updated resource limits, sometimes you need to delete the crashing pod, to ensure that change is happen also to the pod level:

`kubectl delete pod -n namespace podname`

After the deletion, the StatefulSet controller schedules a new instance with the updated resource limits, what's should come up in the Running state:

Verify the pod status by running below command

`kubectl get pod -n namespace -l podname`

