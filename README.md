### Deploy MS in EKS
    eksctl create cluster
    #Does not run well on minikube, use K8S cluster
    kubectl apply -f ./config-microservices.yaml
    #Not needed when using above microservices
    kubectl apply -f devops-monitoring.yml

### OPTIONAL for Linode
    chmod 400 ~/Downloads/online-shop-kubeconfig.yaml
    export KUBECONFIG=~/Downloads/online-shop-kubeconfig.yaml

### On minikube
    // minikube start --cpus 4 --memory 8192 --vm-driver hyperkit
    minikube start --cpus 4 --memory 6144 --extra-config=kubelet.housekeeping-interval=10s
    minikube addons enable metrics-server
    minikube addons enable ingress

### Deploy Prometheus Operator Stack
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    kubectl create namespace monitoring
    helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
    # Install chart with fixed version
    # helm install prometheus prometheus-community/kube-prometheus-stack --version "9.4.1" 
    helm ls

[Link to the chart: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack]

### Check Prometheus Stack Pods
    kubectl get all -n monitoring
    kubectl get crd -n monitoring

### Access Prometheus UI
    kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring &

### Access Grafana
    kubectl port-forward svc/monitoring-grafana 8080:80 -n monitoring &
    user: admin
    pwd: prom-operator

    ps aux | grep kubectl OR ps -ef|grep port-forward #List all the port forwards
    kill -9 2013886


### Trigger CPU spike with many requests

##### Deploy a busybox pod so we can curl our application 
    kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm

##### create a script which curls the application endpoint. The endpoint is the external loadbalancer service endpoint
    for i in $(seq 1 10000)
    do
      curl ae4aee0715edc46b988c6ce67121bf57-1459479566.eu-west-3.elb.amazonaws.com > test.txt
    done

OR
#### Create cpu stress using cpustress from Dockerhub
    kubectl delete pod cpu-test; kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief # "--" is used to pass the parameters to cpustress app.

### Access Alert manager UI
    kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093 &
    AM config: kubectl get secrets alertmanager-monitoring-kube-prometheus-alertmanager-generated -n monitoring -o yaml | less
    echo <Secret> | base64 -D | less

#### Create custom rules:
    Steps to setup alerting and notifications:
    1. Create alerting rules in Prometheus using PrometheusRule CRD
    2. Define routes, matchers and receivers using AlertmanagerConfig CRD 
    3. Check security policies of client, e.g. Gmail: use https://myaccount.google.com/lesssecureapps OR app-password (https://myaccount.google.com/apppasswords) if 2FA enabled. 

    CRD -> PrometheusRule: Refer alert-rules.yaml.
    Syntax: https://docs.openshift.com/container-platform/4.13/rest_api/monitoring_apis/prometheusrule-monitoring-coreos-com-v1.html
    kubectl apply -f alert-rules.yaml
    kubectl get PrometheusRule -n monitoring
    #Check if rules are picked up by Prometheus 
    kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring
    Verify in prometheus UI now.

    CRD -> AlertmanagerConfig: Refer alert-manager-configuration.yaml.
    kubectl apply -f email-secret.yaml
    kubectl apply -f alert-manager-configuration.yaml
    #Check logs of Alertmanager POD and see if new config picked up by Prometheus.

    localhost:9093/api/v2/alerts  # List all the alerts received in AM.


### Deploy Redis Exporter
    Ref's:
    https://github.com/prometheus-community/helm-charts/tree/main/charts
    - https://github.com/oliver006/redis_exporter

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add stable https://charts.helm.sh/stable
    helm repo update

    Create redis-values.yaml (redis-values-local.yaml when installing own redis) to overide default values from helm chart.
    - Enable serviceMonitor.
    - redisAddress
    
    helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml
    helm ls
    kubectl get servicemonitor
    
    #For local setup, not needed if using microservices
    kubectl apply -f redis-deploy.yaml
    curl -v telnet://localhost:6379

    Ref: https://samber.github.io/awesome-prometheus-alerts/
    kubectl apply -f redis-rules.yaml

    Config Grafana dashboard:
    https://grafana.com/grafana/dashboards/763-redis-dashboard-for-prometheus-redis-exporter-1-x/
    import in grafana.

  ### Using client library - to monitor own app
    cd nodejs-app-monitoring-nan-dbc
    cd app
    npm install
    node server.js
    
    # "prom-client": "13.1.0" #Prometheus client lib
    http://localhost:3000/metrics

    #Build docker image 
    docker build -t pndrns/demo-app:nodeapp .
    docker push pndrns/demo-app:nodeapp

    kubectl apply -f k8s-config.yaml
    kubectl port-forward svc/nodeapp 3000:3000
    http://localhost:3000/metrics

    Explore metrics in Prometheus and Create Grafana Dashboard:
    rate(http_request_operations_total[2m]) #Total http request per second over the period of 2 mins
    rate(http_request_duration_seconds_sum[2m]) #Total duration of request per second



    
