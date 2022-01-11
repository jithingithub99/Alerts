### Alert ###

#### KubePersistentVolumeUsageCritical #####
#### KubePersistentVolumeFullInFourDays #####


This alert is due to  low disk space issue in the pvc.We will have to increase the PVC size

First get the pvc details.

` kubectl get pvc pvcname `

` kubectl describe pvc pvcname `

Verify which storage class it belongs to.Confirm "AllowVolumeExpansion" is set to true

` kubectl describe sc storageclassname `

check whether this pod is controlled by any replciaset or deployment

` kubectl describe pod podname `

` kubectl describe deployment deploymentname ` (run if the pod is controlled by deployment)

scale down the replica of deployment to 0

` kubectl scale deployment deploymentname --replicas=0 -n namespace `

Now edit the pvc configuration file

` kubectl edit pvc pvcname `

 resources:
    requests:
      storage: 30Gi > increase to whatever required size

Now after some time we can see PVC size got increased

verify with kubectl get pvc pvcname

Now scale up the deployment replca to earlier value

` kubectl scale deployment deploymentname --replicas=0 -n namespace `

verify pod is running fine
