# Monitor nginx with prometheus and grafana inside kubernetes cluster
## Step 1: Install k3s

* To download k3s on an Ubuntu machine, you can use the following command:
  ```
  curl -sfL https://get.k3s.io | sh -
  ```
  This command downloads a script from the official k3s website and runs it, which will install k3s on your Ubuntu system.
  Check that k3s is running with:
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




