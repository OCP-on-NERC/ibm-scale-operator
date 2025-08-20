# ibm-scale-operator
This repository contains manifests and instructions for installing and configuring ibm-scale operator in MOC environments. The following docs will install the operator version v5.2.3 on OCP 4.18.

## Prerequisites 
1. OCP cluster v4.18
2. mco operator installed
3. ibm software entitlement key obtained
4. ESI networks with routing to ibm-scale storage backend configured and attached to worker nodes
5. remote IBM Storage Scale storage cluster provisioned and configured for ESI

The Official IBM Scale Operator documentation can be found here: [Docs](https://www.ibm.com/docs/en/scalecontainernative/5.2.3)

## Installing ibm-scale for OCP using [host networking](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=planning-deployment-considerations#host-network)

1. Configure worker Nodes
Install machine config on all workers:
```
  kubectl apply -f https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-container-native/v5.2.3.x/generated/scale/mco/ocp4.18/mco_x86_64.yaml
```

Applying the previous command will cause machineConfigPools to reconfigure, enseure they are in a healthy state before continuing: `oc get MachineConfigPool`

2. Apply ibm-scale Operator Manifests
To install the operator and resources run:
```
kubectl apply -f https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-container-native/v5.2.3.x/generated/scale/install.yaml
```

In order for the operator pods to pull their respective images, you need to embed your entitlement key in a secret in each operator namespace:
- ibm-spectrum-scale-operator
- ibm-spectrum-scale-dns
- ibm-spectrum-scale-csi
- ibm-spectrum-scale

See (TODO insert manifests to do this in repo)

3. Verify Operator Installation
To see new operator namespaces: 
```
kubectl get namespaces | grep ibm-spectrum-scale
```

Confirm operator controller manager is running:
```
 kubectl get pods -n ibm-spectrum-scale-operator
```

Confirm operator csi manager is running:
```
 kubectl get pods -n ibm-spectrum-scale-csi
```

4. Apply operator CRs

- Create [cluster](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=resources-cluster) CR: 
```
apiVersion: scale.spectrum.ibm.com/v1beta1
kind: Cluster
metadata:
  labels:
    app.kubernetes.io/instance: ibm-spectrum-scale
    app.kubernetes.io/name: cluster
  name: ibm-spectrum-scale
  namespace: ibm-spectrum-scale
spec:
  daemon:
    # -------------------------------------------------------------------------------
    # hostAliases is optional
    # -------------------------------------------------------------------------------
    # hostAliases is used in an environment where DNS cannot resolve the remote (storage) cluster
    hostAliases:
     - hostname: /* remote hostname */
       ip: /*  remote host IP */
    # Add all ibm storage cluster IPs and hostnames like the above example
    clusterProfile:
      controlSetxattrImmutableSELinux: "yes"
      enforceFilesetQuotaOnRoot: "yes"
      ignorePrefetchLUNCount: "yes"
      initPrefetchBuffers: "128"
      maxblocksize: 16M
      prefetchPct: "25"
      prefetchTimeout: "30"
    # -------------------------------------------------------------------------------
    # nodeSelector is a User Configurable field.
    # -------------------------------------------------------------------------------
    # In conjunction with the nodeSelector configuration, the operator also
    # applies node affinity according to supported architectures and OS.
    # More info on node selectors:
    #       https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector
    nodeSelector:
      scale.spectrum.ibm.com/daemon-selector: ""
    # -------------------------------------------------------------------------------
    # roles is required
    # -------------------------------------------------------------------------------
    # roles is used to set the cluster configuration parameters
    #   applied to all nodes
    #   - during initial cluster creation
    #   - and when changing these parameters at later time
    roles:
    - name: client
      resources:
        cpu: "2"
        memory: 4Gi
  # -------------------------------------------------------------------------------
  # grafana bridge is optional
  # -------------------------------------------------------------------------------
  # Uncomment the grafanaBridge field to enable
  # grafanaBridge: {}
  # -------------------------------------------------------------------------------
  # User must accept the Spectrum Scale license to deploy a CNSA cluster.
  # By specifying "accept: true" below, user agrees to the terms and conditions set
  # forth by the IBM Spectrum Scale Container Native Data Access/Data Management license located
  # at https://www.ibm.com/support/customer/csol/terms/?id=L-WWVS-K7K7DR
  #
  # Enter either data-access or data-management to the license.license field. Customers entitled to
  # the Data Management Edition can use either data-management or data-access. Customers entitled to
  # the Data Access Edition can only use data-access.
  # -------------------------------------------------------------------------------
  license:
    accept: true
    license: data-access
  # -------------------------------------------------------------------------------
  # networkPolicy enabled by default
  # -------------------------------------------------------------------------------
  networkPolicy: {}
```

- Create [remoteCluster](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=resources-remotecluster) CR: 
The following CR is used to configure connection to the remote ibm-scale storage cluster: 
```
apiVersion: scale.spectrum.ibm.com/v1beta1
kind: RemoteCluster
metadata:
  labels:
    app.kubernetes.io/instance: ibm-spectrum-scale
    app.kubernetes.io/name: cluster
  name: remotecluster-sample
  namespace: ibm-spectrum-scale
spec:
  # contactNodes are optional and provides a list of nodes from the storage cluster
  # to be used as the remote cluster contact nodes.  The names should be the daemon
  # node names.  If not specified, the operator will use any 3 nodes detected
  # from the storage cluster.
  contactNodes:
  - /* ess node IP */
  - /* ess node IP */
  gui:
    cacert: cacert-storage-cluster-1
    # This is the secret that contains the CSIAdmin user
    # credentials in the ibm-spectrum-scale-csi namespace.
    csiSecretName: csi-remote-mount-storage-cluster-1
    # hosts are the the GUI endpoints from the storage cluster. Multiple
    # hosts (up to 3) can be specified to ensure high availability of GUI.
    hosts:
    - /* ems IP address */
    # - guihost2.example.com
    # - guihost3.example.com
    insecureSkipVerify: true
    # This is the secret that contains the ContainerOperator user
    # credentials in the ibm-spectrum-scale namespace.
    secretName: cnsa-remote-mount-storage-cluster-1

```

- Create Remote [Filesystem](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=systems-remote-file-system) CR:
```
apiVersion: scale.spectrum.ibm.com/v1beta1
kind: Filesystem
metadata:
  name: essfs-remote
  namespace: ibm-spectrum-scale
  labels:
    app.kubernetes.io/instance: ibm-spectrum-scale
    app.kubernetes.io/name: cluster
spec:
  remote:
    cluster: remotecluster-sample 
    fs: essfs
```

## Verifying ibm-scale operator Install 

See [Verifying the IBM Storage Scale container native cluster
](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=installation-verifying-storage-scale-container-native-cluster)

For some debugging guidance see: [Troubleshooting deployment
](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=troubleshooting-deployment)