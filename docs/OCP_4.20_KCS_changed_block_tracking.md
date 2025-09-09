
# How to use Changed Block Tracking (Dev Preview) in OpenShift 4.20

## Environment

* Red Hat OpenShift Container Platform (OCP) 4.20 and above.


## Issue

OCP 4.20 introduces changed block tracking as a Developer Preview feature. Changed block tracking enables efficient and incremental backups and disaster recovery for persistent volumes (PVs) managed by Container Storage Interface (CSI) drivers. It allows consumers to requests a list of blocks that have changed between two snapshots, which is useful for backup solutions vendors. By only backing up changed blocks, rather than entire volumes, back up processes are more efficient. This KCS demonstrates how developers can begin testing this feature with CSI drivers that support it.


## Resolution

### Prerequisites

The CSI driver must support the [PluginCapability_Service_SNAPSHOT_METADATA_SERVICE](https://github.com/container-storage-interface/spec/blob/f437b19650aaa1a0be79c71770655b591b2d15cd/lib/go/csi/csi.pb.go#L109-L115) capability.
OCP 4.20 does not include any CSI drivers that support this capability by default, so this example deploys [csi-driver-host-path](https://github.com/kubernetes-csi/csi-driver-host-path), which was the first CSI driver to support it.

### Set DevPreviewNoUpgrade on the cluster

This feature is only available on development clusters. To enable it, set the `DevPreviewNoUpgrade` feature set.

**This step will make the cluster unupgradable.**

```
$ oc patch featuregate cluster --type=merge -p '{"spec":{"featureSet":"DevPreviewNoUpgrade"}}'
```

After applying this feature set, a new Custom Resource Definition (CRD) will be created on the cluster:

```
$ oc get crd | grep -i snapshotmetadataservice
snapshotmetadataservices.cbt.storage.k8s.io                       2025-06-24T17:55:55Z
```

The `csi-external-snapshot-metadata` sidecar image is included in the release payload:

```
$ oc adm release info | grep csi-external-snapshot-metadata
  csi-external-snapshot-metadata                 sha256:4a0f46f8a59cdf4a2b92ed02b298788863fb69e7eef44fe6e8aa55e47e763913
```

### Create ClusterRoles

Create two new ClusterRoles: `external-snapshot-metadata-client-runner` to run client tools and `external-snapshot-metadata-runner` for the sidecar.

```
$ oc apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-snapshot-metadata-client-runner
rules:
- apiGroups:
  - snapshot.storage.k8s.io
  resources:
  - volumesnapshots
  - volumesnapshotcontents
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - cbt.storage.k8s.io
  resources:
  - snapshotmetadataservices
  verbs:
  - get
  - list
- apiGroups:
  - ""
  resources:
  - serviceaccounts/token
  verbs:
  - create
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-snapshot-metadata-runner
rules:
- apiGroups:
  - cbt.storage.k8s.io
  resources:
  - snapshotmetadataservices
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
  - get
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
  - get
- apiGroups:
  - snapshot.storage.k8s.io
  resources:
  - volumesnapshots
  - volumesnapshotcontents
  verbs:
  - get
  - list
EOF
```

### Create Service and SnapshotMetadataService

Create the Service first:
```
$ oc apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: csi-snapshot-metadata
  namespace: openshift-cluster-csi-drivers
  labels:
    app.kubernetes.io/name: csi-snapshot-metadata
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: csi-snapshot-metadata-certs
spec:
  ports:
  - name: snapshot-metadata
    port: 6443
    protocol: TCP
    targetPort: 50051
  selector:
    app.kubernetes.io/name: csi-snapshot-metadata
EOF
```

The `service.beta.openshift.io/serving-cert-secret-name` annotation on the Service object generates a new certificate for this service and stores it in a secret. Make sure the `csi-snapshot-metadata-certs` secret was created, since this will be needed by the sidecar:
```
$ oc get secrets -n openshift-cluster-csi-drivers csi-snapshot-metadata-certs
NAME                          TYPE                DATA   AGE
csi-snapshot-metadata-certs   kubernetes.io/tls   2      17s
```

Extract the certificate from the secret and copy it into a new SnapshotMetadataService manifest locally:
```
$ cat | sed "s/GENERATED_CA_CERT/$(oc get secrets -n openshift-cluster-csi-drivers csi-snapshot-metadata-certs -o json | jq -r '.data["tls.crt"]')/" > snapshotmetadataservice.yaml <<EOF
apiVersion: cbt.storage.k8s.io/v1alpha1
kind: SnapshotMetadataService
metadata:
  name: hostpath.csi.k8s.io
spec:
  address: csi-snapshot-metadata.openshift-cluster-csi-drivers.svc:6443
  caCert: GENERATED_CA_CERT
  audience: 005e2583-91a3-4850-bd47-4bf32990fd00
EOF
```

Apply the SnapshotMetadataService manifest to the cluster:
```
$ oc apply -f snapshotmetadataservice.yaml
```

### Create CSIDriver, StorageClass, and VolumeSnapshotClass

```
$ oc apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: hostpath.csi.k8s.io
spec:
  volumeLifecycleModes:
  - Persistent
  - Ephemeral
  podInfoOnMount: true
  fsGroupPolicy: File
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-hostpath-sc
provisioner: hostpath.csi.k8s.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
EOF
```

### Create ServiceAccount and ClusterRoleBindings for the CSI driver

```
$ oc apply -f - <<EOF
kind: ServiceAccount
apiVersion: v1
metadata:
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
  labels:
    app.kubernetes.io/instance: hostpath.csi.k8s.io
    app.kubernetes.io/part-of: csi-driver-host-path
    app.kubernetes.io/name: csi-hostpathplugin
    app.kubernetes.io/component: serviceaccount
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: csi-hostpathplugin-privileged-role
rules:
- apiGroups:
  - security.openshift.io
  resourceNames:
  - privileged
  resources:
  - securitycontextconstraints
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-hostpathplugin-privileged-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: csi-hostpathplugin-privileged-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: hostpath.csi.k8s.io
    app.kubernetes.io/part-of: csi-driver-host-path
    app.kubernetes.io/name: csi-hostpathplugin
    app.kubernetes.io/component: attacher-cluster-role
  name: csi-hostpathplugin-attacher-cluster-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-main-attacher-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: hostpath.csi.k8s.io
    app.kubernetes.io/part-of: csi-driver-host-path
    app.kubernetes.io/name: csi-hostpathplugin
    app.kubernetes.io/component: provisioner-cluster-role
  name: csi-hostpathplugin-provisioner-cluster-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-main-provisioner-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: hostpath.csi.k8s.io
    app.kubernetes.io/part-of: csi-driver-host-path
    app.kubernetes.io/name: csi-hostpathplugin
    app.kubernetes.io/component: resizer-cluster-role
  name: csi-hostpathplugin-resizer-cluster-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-main-resizer-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: hostpath.csi.k8s.io
    app.kubernetes.io/part-of: csi-driver-host-path
    app.kubernetes.io/name: csi-hostpathplugin
    app.kubernetes.io/component: snapshotter-cluster-role
  name: csi-hostpathplugin-snapshotter-cluster-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-main-snapshotter-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: hostpath.csi.k8s.io
    app.kubernetes.io/part-of: csi-driver-host-path
    app.kubernetes.io/name: csi-hostpathplugin
    app.kubernetes.io/component: snapshot-metadata-cluster-role
  name: csi-hostpathplugin-snapshot-metadata-cluster-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-snapshot-metadata-runner
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-hostpathplugin-provisioner-configmap-and-secret-reader-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-provisioner-configmap-and-secret-reader-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-hostpathplugin-provisioner-volumeattachment-reader-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-provisioner-volumeattachment-reader-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-hostpathplugin-provisioner-volumeattributesclass-reader-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-provisioner-volumeattributesclass-reader-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-hostpathplugin-provisioner-volumesnapshot-reader-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-provisioner-volumesnapshot-reader-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-hostpathplugin-resizer-infrastructure-reader-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-resizer-infrastructure-reader-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-hostpathplugin-resizer-storageclass-reader-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-resizer-storageclass-reader-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: csi-hostpathplugin-resizer-volumeattributesclass-reader-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-csi-resizer-volumeattributesclass-reader-role
subjects:
- kind: ServiceAccount
  name: csi-hostpathplugin-sa
  namespace: openshift-cluster-csi-drivers
EOF
```

### Deploy CSI driver with the external-snapshot-metadata sidecar

Create a manifest named `csi-hostpathplugin.yaml` with the following contents:
```
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: csi-hostpathplugin
  namespace: openshift-cluster-csi-drivers
  labels:
    app.kubernetes.io/instance: hostpath.csi.k8s.io
    app.kubernetes.io/part-of: csi-driver-host-path
    app.kubernetes.io/name: csi-snapshot-metadata
    app.kubernetes.io/component: plugin
spec:
  serviceName: "csi-snapshot-metadata"
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: hostpath.csi.k8s.io
      app.kubernetes.io/part-of: csi-driver-host-path
      app.kubernetes.io/name: csi-snapshot-metadata
      app.kubernetes.io/component: plugin
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: hostpath.csi.k8s.io
        app.kubernetes.io/part-of: csi-driver-host-path
        app.kubernetes.io/name: csi-snapshot-metadata
        app.kubernetes.io/component: plugin
    spec:
      serviceAccountName: csi-hostpathplugin-sa
      containers:
        - name: hostpath
          image: registry.k8s.io/sig-storage/hostpathplugin:v1.17.0
          args:
            - "--drivername=hostpath.csi.k8s.io"
            - "--v=5"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--nodeid=$(KUBE_NODE_NAME)"
            - "--enable-snapshot-metadata"
          env:
            - name: CSI_ENDPOINT
              value: unix:///csi/csi.sock
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
          securityContext:
            privileged: true
          ports:
          - containerPort: 9898
            name: healthz
            protocol: TCP
          livenessProbe:
            failureThreshold: 5
            httpGet:
              path: /healthz
              port: healthz
            initialDelaySeconds: 10
            timeoutSeconds: 3
            periodSeconds: 2
          volumeMounts:
            - mountPath: /csi
              name: socket-dir
            - mountPath: /var/lib/kubelet/pods
              mountPropagation: Bidirectional
              name: mountpoint-dir
            - mountPath: /var/lib/kubelet/plugins
              mountPropagation: Bidirectional
              name: plugins-dir
            - mountPath: /csi-data-dir
              name: csi-data-dir
            - mountPath: /dev
              name: dev-dir

        - name: node-driver-registrar
          image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.13.0
          args:
            - --v=5
            - --csi-address=/csi/csi.sock
            - --kubelet-registration-path=/var/lib/kubelet/plugins/csi-hostpath/csi.sock
          securityContext:
            privileged: true
          env:
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
          volumeMounts:
          - mountPath: /csi
            name: socket-dir
          - mountPath: /registration
            name: registration-dir
          - mountPath: /csi-data-dir
            name: csi-data-dir

        - name: liveness-probe
          volumeMounts:
          - mountPath: /csi
            name: socket-dir
          image: registry.k8s.io/sig-storage/livenessprobe:v2.15.0
          args:
          - --csi-address=/csi/csi.sock
          - --health-port=9898

        - name: csi-attacher
          image: registry.k8s.io/sig-storage/csi-attacher:v4.8.0
          args:
            - --v=5
            - --csi-address=/csi/csi.sock
          securityContext:
            privileged: true
          volumeMounts:
          - mountPath: /csi
            name: socket-dir

        - name: csi-provisioner
          image: registry.k8s.io/sig-storage/csi-provisioner:v5.2.0
          args:
            - -v=5
            - --csi-address=/csi/csi.sock
            - --feature-gates=Topology=true
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /csi
              name: socket-dir

        - name: csi-resizer
          image: registry.k8s.io/sig-storage/csi-resizer:v1.13.1
          args:
            - -v=5
            - -csi-address=/csi/csi.sock
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /csi
              name: socket-dir

        - name: csi-snapshotter
          image: registry.k8s.io/sig-storage/csi-snapshotter:v8.2.0
          args:
            - -v=5
            - --csi-address=/csi/csi.sock
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /csi
              name: socket-dir

        - name: csi-snapshot-metadata
          image: quay.io/openshift/origin-csi-external-snapshot-metadata:latest
          command:
          args:
          - -v=5
          - --csi-address=/csi/csi.sock
          - --tls-cert=/tmp/certificates/tls.crt
          - --tls-key=/tmp/certificates/tls.key
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /csi
              name: socket-dir
            - name: csi-snapshot-metadata-server-certs
              mountPath: /tmp/certificates
              readOnly: true

      volumes:
        - hostPath:
            path: /var/lib/kubelet/plugins/csi-hostpath
            type: DirectoryOrCreate
          name: socket-dir
        - hostPath:
            path: /var/lib/kubelet/pods
            type: DirectoryOrCreate
          name: mountpoint-dir
        - hostPath:
            path: /var/lib/kubelet/plugins_registry
            type: Directory
          name: registration-dir
        - hostPath:
            path: /var/lib/kubelet/plugins
            type: Directory
          name: plugins-dir
        - hostPath:
            # 'path' is where PV data is persisted on host.
            path: /var/lib/csi-hostpath-data/
            type: DirectoryOrCreate
          name: csi-data-dir
        - hostPath:
            path: /dev
            type: Directory
          name: dev-dir
        - name: csi-snapshot-metadata-server-certs
          secret:
            secretName: csi-snapshot-metadata-certs
```

Apply the manifest and make sure the pod is running:
```
$ oc apply -f csi-hostpathplugin.yaml
statefulset.apps/csi-hostpathplugin configured

$ oc get pods -n openshift-cluster-csi-drivers csi-hostpathplugin-0
NAME                   READY   STATUS    RESTARTS   AGE
csi-hostpathplugin-0   8/8     Running   0          17s
```

### Create a test PVC, Pod, and VolumeSnapshot

Create a new Namespace:
```
$ oc create namespace testns
```

Create a PVC:
```
$ oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
  namespace: testns
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-hostpath-sc
EOF
```

Create an application pod:
```
$ oc apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-raw
  namespace: testns
  labels:
    name: busybox-test
spec:
  restartPolicy: Always
  containers:
    - image: gcr.io/google_containers/busybox
      command:
      - /bin/sh
      - -c
      - "tail -f /dev/null"
      name: busybox
      volumeDevices:
        - name: vol
          devicePath: /dev/loop3
      securityContext:
        allowPrivilegeEscalation: false
        privileged: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
          - ALL
  volumes:
    - name: vol
      persistentVolumeClaim:
        claimName: csi-pvc
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
EOF
```

Wait for pod to start:
```
$ oc get pods -n testns pod-raw
NAME      READY   STATUS    RESTARTS   AGE
pod-raw   1/1     Running   0          48s
```

Write some data to the volume:
```
$ oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=1 conv=notrunc
$ oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=3 conv=notrunc
$ oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=5 conv=notrunc
$ oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=7 conv=notrunc
$ oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=1 seek=9 conv=notrunc
```

Create the first VolumeSnapshot:
```
$ oc apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot1
  namespace: testns
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: csi-pvc
EOF
```

Wait for it to be ready
```
$ oc get volumesnapshot -n testns
NAME             READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS            SNAPSHOTCONTENT                                    CREATIONTIME   AGE
test-snapshot1   true         csi-pvc                             1Gi           csi-hostpath-snapclass   snapcontent-d9e502dd-aea5-4917-b6fd-7b575087a3e9   2m             2m
```

### Use snapshot-metadata-lister to see the allocated blocks

Create ServiceAccount, ClusterRoleBinding, and Pod for snapshot-metadata-lister:
```
$ oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-metadata-tools-sa
  namespace: testns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: snapshot-metadata-tools-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-snapshot-metadata-client-runner
subjects:
- kind: ServiceAccount
  name: snapshot-metadata-tools-sa
  namespace: testns
---
apiVersion: v1
kind: Pod
metadata:
  name: snapshot-metadata-tools
  namespace: testns
spec:
  serviceAccountName: snapshot-metadata-tools-sa
  initContainers:
  - name: install-client
    image: golang:1.23.4
    command:
    - /bin/sh
    - -c
    - go install github.com/kubernetes-csi/external-snapshot-metadata/examples/snapshot-metadata-lister@latest
    env:
    - name: GOCACHE
      value: "/tmp/"
    - name: GOBIN
      value: "/output/"
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: shared-volume
      mountPath: /output
  containers:
  - name: tools
    image: busybox:1.37.0
    command:
    - /bin/sh
    - -c
    - "tail -f /dev/null"
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: shared-volume
      mountPath: /tools
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - name: shared-volume
    emptyDir: {}
EOF
```

Make sure the pod is running:
```
$ oc get pods -n testns snapshot-metadata-tools
NAME                      READY   STATUS    RESTARTS   AGE
snapshot-metadata-tools   1/1     Running   0          57s
```

Use snapshot-metadata-lister to view allocated blocks in the first snapshot:
```
$ oc exec -n testns -c tools snapshot-metadata-tools -- /tools/snapshot-metadata-lister -n testns -s test-snapshot1
Record#   VolCapBytes  BlockMetadataType   ByteOffset     SizeBytes
------- -------------- ----------------- -------------- --------------
      1     1073741824      FIXED_LENGTH           4096           4096
      1     1073741824      FIXED_LENGTH          12288           4096
      1     1073741824      FIXED_LENGTH          20480           4096
      1     1073741824      FIXED_LENGTH          28672           4096
      1     1073741824      FIXED_LENGTH          36864           4096
```

### Check diff between two snapshots

Write some more data:
```
$ oc exec -n testns pod-raw -- dd if=/dev/urandom of=/dev/loop3 bs=4K count=5 seek=15 conv=notrunc
```

Create another snapshot:
```
$ oc apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot2
  namespace: testns
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: csi-pvc
EOF
```

Wait for it to be ready:
```
$ oc get volumesnapshot -n testns
NAME             READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS            SNAPSHOTCONTENT                                    CREATIONTIME   AGE
test-snapshot1   true         csi-pvc                             1Gi           csi-hostpath-snapclass   snapcontent-c07dccac-3031-402d-a718-8643a8abba87   3m46s          3m46s
test-snapshot2   true         csi-pvc                             1Gi           csi-hostpath-snapclass   snapcontent-61be4ca0-f51a-42e8-ae50-068afcaac614   9s             9s
```

Use snapshot-metadata-lister to see the incremental changes:
```
$ oc exec -n testns -c tools snapshot-metadata-tools -- /tools/snapshot-metadata-lister -n testns -p test-snapshot1 -s test-snapshot2
Record#   VolCapBytes  BlockMetadataType   ByteOffset     SizeBytes
------- -------------- ----------------- -------------- --------------
      1     1073741824      FIXED_LENGTH          61440           4096
      1     1073741824      FIXED_LENGTH          65536           4096
      1     1073741824      FIXED_LENGTH          69632           4096
      1     1073741824      FIXED_LENGTH          73728           4096
      1     1073741824      FIXED_LENGTH          77824           4096
```

### References

* https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/3314-csi-changed-block-tracking/README.md
* https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/docs/deploy-1.17-and-later.md
* https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/docs/example-snapshot-metadata.md
* https://github.com/kubernetes-csi/external-snapshot-metadata/blob/main/deploy/README.md
* https://github.com/kubernetes-csi/external-snapshot-metadata/blob/main/deploy/example/csi-driver/README.md
* https://github.com/kubernetes-csi/external-snapshot-metadata/blob/main/deploy/example/backup-app/README.md
* https://github.com/kubernetes-csi/external-snapshot-metadata/blob/main/client/apis/snapshotmetadataservice/v1alpha1/types.go

