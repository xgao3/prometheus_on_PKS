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

* Install Prometheus Operator using helm chart

    `helm install stable/prometheus-operator --name prometheus --namespace monitoring`  
    
    make sure following repo is available if helm install fails     
    `root@cli-vm:~/helm_rback# helm repo list`    
    `NAME    URL`  
    `stable  https://kubernetes-charts.storage.googleapis.com`
    

     
* If you prefer service type loadbalancer to reach Grafana, edit the service grafana and replace service Type from ‘ClusterIp’ to ‘LoadBalancer’
 
	`kubectl edit service prometheus-grafana -n monitoring`
 
* Find the external IP address of the ‘grafana’ service
	`kubectl get svc -n monitoring | grep grafana | awk '{print $4}' | awk -F , '{print $1}'`
 
* Access `http://<external_IP>` from the web console - Default Username/Password: admin/prom-operator.
