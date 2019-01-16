# prometheus_on_PKS
Instructions to get Prometheus ployed on Kubernetes Cluster operated by PKS

install Prometheus on PKS ( assumption - NSX T Load Balancers are available for type LoadBalancer)


* Ensure you have service account tiller added and cluster role binding updated
 
	· To add, you can copy and paste below YAML into a file (helm_tiller.yaml)
	```yaml\apiVersion: v1
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
 
  Kubectl apply -f helm_tiller.yaml
  or 
  
  kubectl create serviceaccount --namespace kube-system tiller
  kubectl create clusterrolebinding tiller-clusterrolebinding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
  
  kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

* Ensure you have a storage class created by the name 'default', this storage class will be used by the Persistent Volume claims needed for stateful sets.
 
	· To add a storage class store the below YAML file as pks-storageclass.yaml
	```yaml\kind: StorageClass
		apiVersion: storage.k8s.io/v1
		metadata:
		  name: default
		  annotations:
		    storageclass.kubernetes.io/is-default-class: "true"
		provisioner: kubernetes.io/vsphere-volume
		parameters:
		  diskformat: thin
		  
 
	· `Kubectl apply -f pks-storageclass.yaml`
 
* Deploy tiller:

    helm init --upgrade

* Create Monitoring Namespace: 

    kubectl create namespace monitoring

* Install prometheus operator

    helm install coreos/prometheus-operator --name prometheus-operator --namespace monitoring

* Install kube-prometheus

    helm install coreos/kube-prometheus --name kube-prometheus --set global.rbacEnable=true --namespace monitoring

     
* If you prefer service type loadbalancer to reach Grafana, edit the service grafana and replace service Type from ‘ClusterIp’ to ‘LoadBalancer’
 
	`kubectl edit service kube-prometheus-grafana`
 
* Find the external IP address of the ‘grafana’ service
	`kubectl get svc -n monitoring | grep grafana | awk '{print $4}' | awk -F , '{print $1}'`
 
* Access `http://<external_IP>` from the web console
