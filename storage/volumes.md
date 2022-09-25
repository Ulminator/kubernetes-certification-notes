# Volumes

On-disk files in a Container are ephemeral.  
When a Container crashes, kubelet will restart it, but the files will be lost.  
When running Containers together in a Pod it is ofter necessary to share files between them.

Difference between Docker volumes and Kubernetes volumes:

**Docker**
* Volumes are simply a directory on disk or in another Container.
* Lifetimes are not managed and until recently were only local-disk-backed volumes.

**Kubernetes**
* Has an explicit lifetime - the same as the Pod that encloses it.
* Consequently, a volume outlives any Containers that run within the Pod, and data is preserved across Container restarts.
* Supports many types of volumes, and a Pod can use any number of them simultaneously.
* Specify volumes for the Pod with `.spec.volumes` field and where to mount with `.spec.containers[*].volumeMounts` field.

Volumes cannot mount onto other volumes or have hard links to other volumes. Each Container in the Pod must independently specify where to mount each volume.

## Persistent Volumes

* A piece of storage in the cluster that is provisioned using Storage Classes.
* Volume plugins like volumes, but have a lifecycle independent of any individual Pod that uses the PV.

A Persistent Volume Claim (PVC) is a request for storage by a user. It is similar to a Pod:

* Pods consume node resources and PVCs consume PV resources.
* Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes.

### Lifecycle of a Volume and Claim

1. Provisioning

There are two ways PVs may be provisioned.

Static
* A cluster admin creates a number of PVs.
* They carry the details of the real storage, which is available for the cluster users.
* They exist in Kubernetes API and are available for consumption.

Dynamic
* When none of the static PVs the administrator created match a user's PVC, the cluster may try to dynamically provision a volume specifically for the PVC.
* Based on the StorageClasses: the PVC must request a storage class and the administrator must have created and configured that class for dynamic provisioning to occur.
* Claims that request the class `""` effectively disable dynamic provisioning themselves.
* To enable dynamic storage provisioning based on storage class, the cluster admin needs to enable the `DefaultStorageClass` admission controller on the API server. This can be done, for example, by ensuring that DefaultStorageClass is among the comma-delimited, ordered list of values for the `--enable-admission-plugins` flag of the API server component. For more information on API server command-line flags, check kube-apiserver documentation.

2. Binding
* A control loop in the master watches for new PVCs, finds a matching PV (if possible), and binds them together.
* If a PV was dynamically provisioned for a new PVC, the loop will bind that PV to the PVC.
* Otherwise, the user will always get at least what they asked for, but the volume may be in excess of what was requested.
* Once bound, PVC binds are exclusive, regardless of how they were bound.
* A PVC to PV binding is a 1-1 mapping, using a ClaimRef which is bi-directional between the PV and PVC.
* Claims will remain unbound indefinitely if matching volume does not exist.
    * Claims will be bound as matching volumes become available.

3. Using
* Pods use claims as volumes.
* The cluster inspects the claim to find the bound volume and mounts that volume for a Pod.
* For volumes that support multiple access modes, the user specifies which mode is desired when using their claim as a volume in a Pod.

4. Storage Object in Use Protection
* The purpose of this feature is to ensure that PVCs in active use by a Pod and PVs that are bound to PVCs are not removed by the system, as this may result in data loss.
    * **NOTE**: PVC is in active use by a Pod when a Pod object exists that is using the PVC.
* If a user deletes a PVC in active use by a Pod, the PVC is not removed immediately.
    * PVC removal is postponed until the PVC is no longer actively used by any Pods.
    * You can see that a PVC is protected when the PVC's status is `Terminating` and the `Finalizers` list includes `kubernetes.io/pvc-protection`.
* If an admin deletes a PV that is bound to a PVC, the PV is not removed immediately.
    * PV removal is postponed until the PV is no longer bound to a PVC.
    * You can see that a PV is protected when the PV's status is `Terminating` and the `Finalizers` list includes `kubernetes.io/pv-protection`.

5. Reclaiming
* When a user is done with their volume, they can delete the PVC objects from the API that allows reclamation of the resource.
* The reclaim policy for a PV tells the cluster what to do with the volume after it has been released of its claim.
* Reclaim Policies
    * Retain
        * Allows for manual reclamation fo the resource.
        * After PVC is deleted, the PV still exists and the volume is considered "released".
        * It is not yet available for another claim because the previous claimant's data remains in the volume.
        * An admin can manually reclaim the volume with the following steps.
            1. Delete the PV. The associated storage asset in external infrastructure still exists after the PV is deleted.
            2. Manually clean up the data on the associated storage asset acordingly.
            3. Manually delete the associated storage asset, or if you want to reuse the same storage asset, create a new PV with the storage asset definition.
    * Delete
        * Deletion removes both the PV object from Kubernetes, as well as the associated storage asset in the external infrastructure.
        * Dynamically provisioned volumes inherit the reclaim policy of their StorageClass, which defaults to `Delete`.
            * The administrator should configure the StorageClass according to user's expectations.
            * Otherwise, the PV must be edited or patched after it is created.
    * Recycle
        * Performs a basic scrub (`rm -rf /thevolume/*`) on the volume and makes it available again for a new claim.
        * An admin can configure a custom recycler Pod template using the Kubernetes controller manager command line arguments as described [here](https://kubernetes.io/docs/admin/kube-controller-manager/).

TODO: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes

## Storage Classes
