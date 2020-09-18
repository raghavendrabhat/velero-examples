# OCS deploy/tuning for OADP
The deployment of OCS in an OCP environment is not any different from the
recommended steps in OCS documentation.

NOTE: The assumption is that OCS is deployed in the namespace (or project) "openshift-storage". Create if it does not exist. It can be created from\
      openshift console.

Navigate to the OpenShift console. Under the Administrator view, go to Operators on the left tab and click on OperatorHub. Search for the OpenShift Container Storage operator in the search bar. Click on it to install and subscribe to the operator.

![OCS OperatorHub](/ocs/images/ocs_operatorhub.png)

If you go to Installed Operators and select the openshift-storage project, you should see the OpenShift Container Storage operator successfully installed:

![OCS Installed](/ocs/images/ocs_installed.png)

The above steps just installs the OCS operator. To setup a storage based out of OCS for application consumption, a storagecluster
has to be created. The below steps mentions the steps to follow for creating a complete OCS cluster that can be consumed as PVs
for not only storage requirements of the application, but also for snapshot requirements by applications like velero.

The next step is to create a storagecluster for providing storage for applications. Note that assumption is that the OCP cluster has
sufficient resources (both the number of nodes and the physical resources within the nodes).

Navigate to Installed Operators and select the openshift-storage project, you should see the OpenShift Container Storage operator successfully installed.
There it has option at the right most side to create a storage cluster (or ocs cluster). Click it.

![Storage Cluster Creation](/ocs/images/storage_cluster_creation.png)

There you will see options as shown in this image.

![Storage Cluster Options](/ocs/images/storage_cluster_options.png)

Ensure that, atleast 3 nodes are available for storage. Select them.
Also, if encryption is enabled by default, turn it off. Otherwise, rook
will face problems. Select the amount of total storage required for the
storage cluster (small, medium or large) and then click create. This should
take some time (few minutes).

Once everything is deployed this is what the console would display to indicate
a successful deployment of OCS.

For getting more details as to what pods are running to provide and manage storage,
the following command would be helpful. It should show pods corresponding to the
underlying storage solution, the CSI plugins deployed, noobaa pods and other pods
such as the rook operator and other operators.
```
oc get pods -n openshift-storage
```

Another useful next step might be to change the default storageclass to make OCS the
default. Otherwise it would be gp2. And to look at the storageclasses that are available
in the system, run the below command. It can also be a method for verifying that the
appropriate storageclasses haves been deployed as part of OCS deployment.
```
oc get storageclass
```

Changing the default storageclass can be done via setting annotations on the
corresponding storageclasses.

Run below command to make gp2 storageclass non-default
```
oc patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

And to make a particular storageclass deployed by OCS default one, run the below command.
```
oc patch storageclass <storageclass name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

And finally, to be able to do backup/restore of applications that have storage provided by OCS, we have to ensure that
CSI snapshots can be taken properly. Two things should be ensured for this.

1) Ensure that snapshot crds and snapshot-controller are there in the cluster.

   In OCP-4.6, the snapshot-controller is deployed in a namespace called "openshift-cluster-storage-operator". Ensure
   that pods are running there for snapshot-controller

2) Ensure that volumesnapshotclass resource is created in the cluster.

   To list the available snapshot classes, run the below command.
   ```
   oc get volumesnapshotclass
   ```

    a) If volumesnapshotclass is not there, then create one. However care must be taken to ensure that the driver field
    in the volumesnapshotclass yaml is same as the provisioner in the storageclass definition.

    This is how a volumesnapshotclass yaml and the corresponding storageclass yaml would look like.
    ```
    ---
    apiVersion: snapshot.storage.k8s.io/v1beta1
    kind: VolumeSnapshotClass
    metadata:
      name: csi-rbdplugin-snapclass
    driver: openshift-storage.rbd.csi.ceph.com
    parameters:
      clusterID: openshift-storage
      csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
      csi.storage.k8s.io/snapshotter-secret-namespace: openshift-storage
    deletionPolicy: Retain
    ```

    The corresponding storageclass (rbd) to which the above snapshotclass relates is as below.
    ```
    allowVolumeExpansion: true
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
      creationTimestamp: "2020-08-21T16:26:06Z"
      managedFields:
      - apiVersion: storage.k8s.io/v1
        fieldsType: FieldsV1
        fieldsV1:
          f:allowVolumeExpansion: {}
          f:parameters:
            .: {}
            f:clusterID: {}
            f:csi.storage.k8s.io/controller-expand-secret-name: {}
            f:csi.storage.k8s.io/controller-expand-secret-namespace: {}
            f:csi.storage.k8s.io/fstype: {}
            f:csi.storage.k8s.io/node-stage-secret-name: {}
            f:csi.storage.k8s.io/node-stage-secret-namespace: {}
            f:csi.storage.k8s.io/provisioner-secret-name: {}
            f:csi.storage.k8s.io/provisioner-secret-namespace: {}
            f:imageFeatures: {}
            f:imageFormat: {}
            f:pool: {}
          f:provisioner: {}
          f:reclaimPolicy: {}
          f:volumeBindingMode: {}
        manager: ocs-operator
        operation: Update
        time: "2020-08-21T16:26:06Z"
      - apiVersion: storage.k8s.io/v1
        fieldsType: FieldsV1
        fieldsV1:
          f:metadata:
            f:annotations:
              .: {}
              f:storageclass.kubernetes.io/is-default-class: {}
        manager: kubectl-patch
        operation: Update
        time: "2020-08-25T14:53:16Z"
      name: ocs-storagecluster-ceph-rbd
      resourceVersion: "7370551"
      selfLink: /apis/storage.k8s.io/v1/storageclasses/ocs-storagecluster-ceph-rbd
      uid: a0bd75a6-1abc-47c6-a278-2372fdc76bc1
      parameters:
      clusterID: openshift-storage
      csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
      csi.storage.k8s.io/controller-expand-secret-namespace: openshift-storage
      csi.storage.k8s.io/fstype: ext4
      csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
      csi.storage.k8s.io/node-stage-secret-namespace: openshift-storage
      csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
      csi.storage.k8s.io/provisioner-secret-namespace: openshift-storage
      imageFeatures: layering
      imageFormat: "2"
      pool: ocs-storagecluster-cephblockpool
    provisioner: openshift-storage.rbd.csi.ceph.com
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    ```

    b) If the volumesnapshotclass exists, still its yaml file requires edition. We have to change the DeletionPolicy
    of the volumesnapshotclass that is being used. By default, the DelitionPolicy is to remove the underlying snapshots
    whenever "volumesnapshot" resource is removed. However, this can create issues whenever a disaster occurs in form of
    namespace deletion. "volumesnapshot" resources are namespaced resources. And deletion of the namespace results in
    removal of "volumesnapshot" resources, which in turn results in deletion of underlying snapshots if the DeletionPolicy
    is "Delete". Hence, to avoid this, the DeletionPolicy of the volumesnapshotclass has to be changed to Retain. Otherwise
    at the time of restoration, the appropriate snapshots from the storage layer would be referenced and absence of them will
    cause a restoration failure.

    ```
    oc edit volumesnapshotclass/<snapshotclass name>
    ```

    However, note that this can result in snapshots piling up in the system. The cluster scoped snapshot resource in
    kubernetes that has direct reference to the underlying storage snapshot is "volumesnapshotcontent" object. Thus, if we
    change the DeletionPolicy to "Retain", then in the kubernetes cluster, there will be piling up of "volumesnapshotcontent"
    objects. And they require a manual deletion.

    ```
    oc delete volumesnapshotcontent/<volumesnapshotcontent object>
    ```

    And sometimes, the deletion of those resouces via above command can hang. And in such situations one needs to remove the
    finalizer manually to ensure deletion of volumesnapshotcontent object.

Finally, once we have reached this stage, the storage is ready for consumption by both stateful applications and backup applications.
