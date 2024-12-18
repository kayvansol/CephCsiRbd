# Ceph csi rbd On Kubernetes Cluster

The Container Storage Interface (CSI) provides a consistent and extensible framework for storage vendors to integrate their solutions with Kubernetes. It enables seamless integration of different storage systems, allowing users to take advantage of the unique features and capabilities of their chosen storage backend.

Kubernetes Ceph CSI is a CSI driver that enables Kubernetes clusters to leverage the power of Ceph for persistent storage management. It simplifies the provisioning and management of Ceph storage for containerized applications, providing a seamless experience for developers and administrators alike. Let’s delve into some of the key benefits and features of Kubernetes Ceph CSI.

You may use Ceph Block Device images with Kubernetes through ceph-csi, which dynamically provisions RBD images to back Kubernetes volumes and maps these RBD images as block devices (optionally mounting a file system contained within the image) on worker nodes running pods that reference an RBD-backed volume. Ceph stripes block device images as objects across the cluster, which means that large Ceph Block Device images have better performance than a standalone server!

To use Ceph Block Devices with Kubernetes, you must install and configure ceph-csi within your Kubernetes environment.

At the Ceph Cluster side :

you can deploy a ceph cluster as the link :

CREATE A POOL
By default, Ceph block devices use the rbd pool. Create a pool for Kubernetes volume storage. Ensure your Ceph cluster is running, then create the pool.

```
ceph osd pool create kubernetes
```
See Create a Pool for details on specifying the number of placement groups for your pools, and Placement Groups for details on the number of placement groups you should set for your pools. A newly created pool must be initialized prior to use. Use the rbd tool to initialize the pool :
```
rbd pool init kubernetes
```

SETUP CEPH CLIENT AUTHENTICATION
Create a new user for Kubernetes and ceph-csi. Execute the following and record the generated key :
```
ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
```

the result is :
```
[client.kubernetes]
    key = AQDS7VVnV2MlAhAAG03XBGjyLpgsOlovWw36OQ==
```

GENERATE CEPH-CSI CONFIGMAP
The ceph-csi requires a ConfigMap object stored in Kubernetes to define the the Ceph monitor addresses for the Ceph cluster. Collect both the Ceph cluster unique fsid and the monitor addresses :
```
ceph mon dump
```

At the Kubernetes Cluster side :

ConfigMap :

Generate a csi-config-map.yaml file similar to the example below, substituting the fsid for “clusterID”, and the monitor addresses for “monitors” :
```
cat <<EOF > csi-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "f374857a-b28f-11ef-97e5-37412429fbe1",
        "monitors": [
          "192.168.56.143:6789",
          "192.168.56.144:6789",
          "192.168.56.145:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
EOF
```

Once generated, store the new ConfigMap object in Kubernetes :
```
kubectl apply -f csi-config-map.yaml
```
Recent versions of ceph-csi also require an additional ConfigMap object to define Key Management Service (KMS) provider details. If KMS isn’t set up, put an empty configuration in a csi-kms-config-map.yaml file or refer to examples at https://github.com/ceph/ceph-csi/tree/master/examples/kms :
```
cat <<EOF > csi-kms-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
EOF
```
Once generated, store the new ConfigMap object in Kubernetes :
```
kubectl apply -f csi-kms-config-map.yaml
```
Recent versions of ceph-csi also require yet another ConfigMap object to define Ceph configuration to add to ceph.conf file inside CSI containers :
```
cat <<EOF > ceph-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  ceph.conf: |
    [global]
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
  # keyring is a required key and its value should be empty
  keyring: |
metadata:
  name: ceph-config
EOF
```

Once generated, store the new ConfigMap object in Kubernetes :
```
kubectl apply -f ceph-config-map.yaml
```

GENERATE CEPH-CSI CEPHX SECRET
ceph-csi requires the cephx credentials for communicating with the Ceph cluster. Generate a csi-rbd-secret.yaml file similar to the example below, using the newly created Kubernetes user id and cephx key :

