Note the below additional configurations while configuring Kubernetes the hard way and setting up metrics-server as per https://github.com/kubernetes-sigs/metrics-server.

Add the below in the kube-apiservice systemd file -

```
  --requestheader-client-ca-file=/var/lib/kubernetes/ca.crt \
  --requestheader-allowed-names="" \  -- This can be 'aggregator' as well if the appropriate cert with the aggregator CN is present.
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --proxy-client-cert-file=/var/lib/kubernetes/kube-api.crt \
  --enable-aggregator-routing=true \
  --proxy-client-key-file=/var/lib/kubernetes/kube-api.key \
```

Update the ***metrics-server-deployment.yaml*** file ```template.spec``` to include the ```command``` array and hostNetwork property as 'true'. Setting hostNetwork as true is a hack required for the weave overlay network in this set up.

**TODO:** Yet to find a better solution.

```
spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        command:
        - /metrics-server
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
        imagePullPolicy: Always
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
      hostNetwork: true
```

```system:kube-apiserver-metrics``` cluster role binding maps the ```system:aggregated-metrics-reader``` cluster role to ```kube-apiserver``` user.  

Refer ```system:aggregated-metrics-reader``` cluster role in ***aggregated-metrics-reader.yaml*** file in the code repo.

```metrics.k8s.io``` API group is defined in ***metrics-apiservice.yaml*** file.

```
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver-metrics
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:aggregated-metrics-reader
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kube-apiserver
 ```
