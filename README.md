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

#### 3) KubePodOOMKilled #####

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

 #### 4) KubeDeploymentReplicasMismatch #####
 
 Replica mismatche commonly happens whenever there is a pod associated with a deployment failes or not working.There could be many reason however we can check pod status and proceed.For these alerts also we can follow the same steps of KubePodOOMKilled issues.

#### 5) VeleroBackupTakesTooLong #####



VeleroBackupTakesTooLong alerts is because of some failed backup job due to various reason or KKP bug  

Login to velero pod and run below command to check the backups

`./velero backup get  |more`

check the status of velero backups whether completed or not 

![image](https://user-images.githubusercontent.com/89779991/149271282-160a87d4-7c53-43e0-bf64-11a8d6ba3648.png)


Check the KKP version, if it is  < 2.18 env we can  delete the partial failed backups

`velero backup list`

`velero backup delete xxxx`

Delete running velero pod, then a fresh one starts and should report the right metrics.

`kubectl delete pod veleropod`

verify all velero backups 

`velero backup list`


#### 6) ThanosSidecarNoHeartbeat #####

This  alert is coming when container Thanos trying to access the pod Prometheus it is getting a connection refused error.

we will have to check pod logs

`kubectl get logs `

```
2021-11-11T10:11:40.912346609Z stderr F level=error ts=2021-11-11T10:11:40.912200097Z 
caller=runutil.go:99 component=reloader msg="function failed. Retrying in next tick" 
err="trigger reload: reload request failed: Post \"http://localhost:9090/-/reload\": dial 
tcp [::1]:9090: connect: connection refused

```

 From the logs we could see that there are 2-3 warnings says health check service sending SIGTERM signal, and the pod exists gracefully.
 
 `2021-11-11 15:40:47 msg="Received SIGTERM, exiting gracefully..." `
 
 ` 2021-11-11 15:40:47 msg="changing probe status" `
 
 Here the pod which was supposed to get terminated was continuing even after the new pod 
creation and this could be reason for no heartbeat from that particular pod for some time (average 
value of 300s) and it gets fixed automatically when the old pod is killed

#### 7) User Cluster API Client disconnects #####

This issue could be caused by:
1. high number of restarts of the API server of the cluster nl2x8h2mpf , seams to less "reserved" cpu / memory.
we can increase cluster component resource limits for this cluster

2. The nodeport-proxy envoy has no resource limits set (is kind of the ingress for all api server calls), this potential
leads also to some delays or connection drops if this pod or node is under pressure and we can't ensure the cpu/memory for it in the current Version.

Therefor we update the PROD environment to an already present bugfix release 2.17.5. As their are only a few bugfix changes in the release and no feature/migration triggered we don't expect any Problem. 
With the 2.18 release of KKP, the nodeport proxy will get an additional anti-affinity to spread the pods across multiple nodes and zones,


##### 8) KubePodNotReady #####

##### 9) PromScrapeFailed #####

##### 10) KubeAPILatencyHigh #####


