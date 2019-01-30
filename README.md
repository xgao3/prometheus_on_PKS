# prometheus_on_PKS
Instructions to get Prometheus deployed on Kubernetes Cluster operated by PKS

install Prometheus on PKS ( assumption - NSX T Load Balancers are available for type LoadBalancer)


* Create a service account for Tiller and bind it to the cluster-admin role by adding the following section to rbac-config.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tiller-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: ""
 
kubectl apply -f rbac-config.yaml
 or 
  
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-clusterrolebinding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
  ```

* Ensure you have a storage class created by the name 'default', this storage class will be used by the Persistent Volume claims needed for stateful sets.
 
· To add a storage class store the below YAML file as pks-storageclass.yaml
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: default
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/vsphere-volume
parameters:
  diskformat: thin  
  
· kubectl apply -f pks-storageclass.yaml
 ```
* Download and install the [Helm CLI](https://github.com/helm/helm/releases)

* Deploy tiller:

    `helm init --service-account tiller`

* Create Monitoring Namespace: 

    `kubectl create namespace monitoring`  

* Install the cert-manager CRDs.

```
    kubectl apply
       -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.6/deploy/manifests/00-crds.yaml
```

* Update helm repository cache

    `helm repo update`

* Install cert-manager
```
helm install \
   --name cert-manager \
   --namespace cert-manager \
   --version v0.6.0 \
   stable/cert-manager
```
* Install nginx ingress controller

    `helm install stable/nginx-ingress --name quickstart`

* Generate cert and upload as secret 
```
    openssl genrsa -out ca.key 2048

    openssl req -x509 -new -nodes -key ca.key \
        -subj "/CN=${COMMON_NAME}" -days 3650 \
        -reqexts v3_req -extensions v3_ca -out ca.crt

    kubectl create secret tls ca-key-pair \
        --cert=ca.crt \
        --key=ca.key \
        --namespace=default
```

* create certficate issuer (name issuer.yaml)

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: default
spec:
  ca:
    secretName: ca-key-pair
    
kubectl create -f issuer.yaml
```

* create certificate issuer (desired-cert.yaml)

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: ca-issuer
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
  commonName: example.com
  organization:
  - Example CA
  dnsNames:
  - example.com
  - www.example.com

kubectl create -f issuer.yaml
```
    
* We are going to create following customization file so data will be written to persistent volume and place Grafana behind an ingress with TLS (custom.yaml)

```yaml
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi

prometheus:
  prometheusSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi

grafana:
  adminPassword: "VMware1!"
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - grafana.test.example.com
    tls:
      - secretName: example-com-tls
        hosts:
          - grafana.test.example.com
  persistence:
    enabled: true
    accessModes: ["ReadWriteOnce"]
    size: 1Gi


* Install Prometheus Operator using helm chart

    `helm install stable/prometheus-operator --name prometheus --namespace monitoring -f custom.yaml`  
    
    make sure following repo is available if helm install fails     
    `root@cli-vm:~/helm_rback# helm repo list`    
    `NAME    URL`  
    `stable  https://kubernetes-charts.storage.googleapis.com`
    
     
* If you want to reach prometheus UI
 
	`kubectl port-forward prometheus-prometheus-oper-prometheus -n monitoring 9090:9090`
 
* Find the external IP address of the ‘grafana’ service
	`kubectl get svc -n monitoring | grep  quickstart-nginx-ingress-controller | awk '{print $4}' | awk -F , '{print $1}'`
	
* Add above entry to /etc/hosts

* Access `https://grafana.test.example.com/` from the web console - Default Username/Password: admin/VMware1!.
