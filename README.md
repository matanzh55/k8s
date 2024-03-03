# Monitor nginx with prometheus and grafana inside kubernetes cluster
## Step 1: Install k3s

* To download k3s on an Ubuntu machine, you can use the following command:
  ```
  curl -sfL https://get.k3s.io | sh -
  ```
  This command downloads a script from the official k3s website and runs it, which will install k3s on your Ubuntu system.
* Check that k3s is running with:
  ```
  sudo k3s kubectl get node
  ```
## Step 2: Deploy Nginx

* Create a deployment for Nginx:
  ```
  sudo k3s kubectl create deployment nginx --image=nginx
  ```
* Expose Nginx via a service so it can be accessed from outside the cluster:
  ```
  sudo k3s kubectl expose deployment nginx --port=80 --type=NodePort
  ```
* Find out what port Nginx is exposed on by running:
  ```
  sudo k3s kubectl get services
  ```
## Step 3: Set Up Prometheus and Grafana
* Install Helm, which is a package manager for Kubernetes that simplifies the installation of applications:
  ```
  curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
  ```
* k3s creates its own Kubeconfig file when it is installed, and you need to specify this file for Helm to interact with the k3s cluster.
  By default, the k3s Kubeconfig file is located at /etc/rancher/k3s/k3s.yaml.
  You can specify the Kubeconfig file for Helm by setting the KUBECONFIG environment variable or by using the --kubeconfig flag with Helm commands.
  Here's how you can set the KUBECONFIG environment variable:
  ```
  export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
  ```
* Add the Prometheus Helm chart repository and install Prometheus:
  ```
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  helm install prometheus prometheus-community/prometheus
  ```
* Add the Grafana Helm chart repository and install Grafana:
  ```
  helm repo add grafana https://grafana.github.io/helm-charts
  helm install grafana grafana/grafana
  ```
## Step 4: Ensure Nginx Exposes Metrics
* For Prometheus to scrape metrics from Nginx, you need an exporter since Nginx does not natively expose Prometheus metrics.
  Create a yaml file "nginx-prometheus-exporter-deployment.yaml" and deploy it with:
  ```
  kubectl apply -f nginx-prometheus-exporter-deployment.yaml
  ```
* Create a service to expose the exporter - Create a yaml file "nginx-prometheus-exporter-service.yaml" and deploy it with:
  ```
    kubectl apply -f nginx-prometheus-exporter-service.yaml
  ```
* Edit the Prometheus ConfigMap:
  ```
  kubectl edit configmap prometheus-server -n default
  ```
* Add a new job to the scrape_configs section of the Prometheus configuration file within the ConfigMap:
  ```
  scrape_configs:
  # ... other scrape jobs ...
  
  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-prometheus-exporter.default.svc.cluster.local:9113']
   ```
  Save and exit the editor.
* If Prometheus doesn't automatically reload its configuration, you might need to restart it:
  ```
  kubectl rollout restart deployment prometheus-server -n default
  ```
* Exec into the running Nginx pod:
  ```
  kubectl exec -it <nginx-pod-name> -n default -- /bin/sh
  ```
* You will need to add the following to your Nginx configuration inside the server block of default.conf:
  ```
  
  location /basic_status {
      stub_status on;
      allow all; # Allow all IPs to access metrics
  }
  ```
* After adjusting the configuration, remember to reload Nginx to apply the changes:
  ```
  nginx -t
  nginx -s reload
  ```
* Restart the Exporter Pod - reapply the deployment and restart the exporter pod:
  ```
  kubectl apply -f nginx-prometheus-exporter-deployment.yaml
  kubectl delete pod <nginx-prometheus-exporter-pod-name> -n default
  ```
* After the exporter pod restarts, monitor its status to ensure it transitions to a Running state without crashing:
  ```
  kubectl get pod <nginx-prometheus-exporter-pod-name> -n default
  ```
* Access Prometheus UI Using Port Forwarding - Open a terminal and run the following command:
  ```
  kubectl port-forward svc/prometheus-server 9090:80
  ```
  Now, you can access the Prometheus UI by navigating to http://localhost:9090 in your web browser.
  
* Once you have the Prometheus UI open, you can check if Prometheus is correctly scraping metrics from Nginx:
  In the Prometheus UI, go to the "Status" menu and select "Targets". This page lists all the scrape targets configured for Prometheus.
  Look for the Nginx target. It should be listed with the job name you configured for scraping metrics from Nginx. Ensure the status is "UP", indicating Prometheus is 
  successfully scraping metrics.

## Step 5: Set Up Grafana Dashboard
* Obtain the Grafana admin password:
  ```
  kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
  ```
* Expose Grafana to access it via the browser:
  ```
  kubectl expose service grafana --type=NodePort --target-port=3000 --name=grafana-np
  ```
  To access Grafana, you'll need to know the NodePort that was assigned to the grafana-np service.
  You can find this by running:
  ```
  kubectl get service grafana-np
  ```
* Look for the port mapping under the PORT(S) column. It will be in the format 80:XXXXX/TCP, where XXXXX is
  the NodePort number through which you can access Grafana.
  You can access Grafana by navigating to http://<Node-IP>:<NodePort> in your web browser, replacing <Node-IP> with your node's IP address and <NodePort> with the actual 
  port number you identified. Access Grafana with: username - admin, password - the password you obtain already.

* Add Prometheus as a Data Source:
  Go to the Configuration menu (gear icon on the left sidebar), then select Data Sources.
  Click Add data source and select Prometheus.
  Enter the URL of your Prometheus server. If Prometheus is running in the same Kubernetes cluster, this will be something like 
  http://prometheus.default.svc.cluster.local.
  Save and test the data source to ensure Grafana can connect to Prometheus.
  
* To create a new dashboard:
 Click on the + icon on the left sidebar and select Create Dashboard.
 Click Add new panel.
 In the query editor, select your Prometheus data source and start typing your query to pull in Nginx metrics, such as nginx_http_requests_total or nginx_http_connections.
 Customize the panel with the visualization type you want (Graph, Gauge, Bar Gauge, etc.).
 Set the panel title and any other settings you want to tweak.

* Continue adding panels for different metrics you are interested in monitoring.



  
  



  
  




