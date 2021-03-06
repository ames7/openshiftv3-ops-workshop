# Deploying Metrics

In this lab you will learn how to deploy metrics. Deployment of metrics need a backend storage for this. Please see either the [Persistant Volume Claim](creating_persistent_volume.md) or the [Container Native Storage](cns.md) labs before doing this lab.

## Step 1

Switch to the `openshift-infra` project

```
oc project openshift-infra
```

Metrics needs a backend storage for the database. You'll need a `pvc` before you proceed. Below is an example using [cns](cns.md) as the backend storage (your `pvc` might differ).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: metrics-storage
 annotations:
  volume.beta.kubernetes.io/storage-class: gluster-block
spec:
 accessModes:
  - ReadWriteOnce
 resources:
   requests:
     storage: 20Gi
```

Create this claim.

```
$ oc create -f metrics-storage-pvc.yaml
persistentvolumeclaim "metrics-storage" created
```

Wait for it to go from "Pending" to "Bound"
```
$ oc get pvc
NAME              STATUS    VOLUME    CAPACITY   ACCESSMODES   STORAGECLASS    AGE
metrics-storage   Pending                                      gluster-block   7s

$ oc get pvc
NAME              STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS    AGE
metrics-storage   Bound     pvc-5ea4435f-c410-11e7-8f74-029fe14a0ff8   20Gi       RWO           gluster-block   3s
```

## Step 2

Using the ansible playbook provided by OpenShift, deploy the metrics stack and make you change these options to what makes sense to you. Take note of the options `openshift_metrics_cassandra_storage_type` and `openshift_metrics_cassandra_pvc_size`. The size must match the claim you had above.

```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/openshift-metrics.yml \
-e openshift_metrics_install_metrics=True \
-e openshift_metrics_hawkular_hostname=hawkular.apps.example.com \
-e openshift_metrics_cassandra_storage_type=pv \
-e openshift_metrics_cassandra_pvc_size=20Gi
```

There is a known issue where you'll have two `pvc`'s show up. Overwrite the the `rc` with the right value. 
```
oc volume rc/hawkular-cassandra-1 --add --overwrite --name=cassandra-data -t pvc --claim-name=metrics-storage
```

And delete the other `pvc`
```
$ oc delete pvc metrics-cassandra-1
persistentvolumeclaim "metrics-cassandra-1" deleted
```

## Conclusion

In this lab you learned how to set up metrics with a backend storage. You should now have metrics collection running.

```
$ oc get pods
NAME                         READY     STATUS    RESTARTS   AGE
hawkular-cassandra-1-08jnc   1/1       Running   0          2m
hawkular-metrics-19vs0       1/1       Running   0          7m
heapster-90862               1/1       Running   0          7m
```
