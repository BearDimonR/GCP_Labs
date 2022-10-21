## **Create and Manage Cloud Resources: Challenge Lab**

### Create Virtual Machine

**Requirements:**

- Name the instance **`Instance name`** .
- Use an *f1-micro* machine type.
- Use the default image type (Debian Linux).

```bash
gcloud compute instances create $INSTANCE_NAME --machine-type=f1-micro
```

### Create Kubernetes Cluster

The team is building an application that will use a service running on Kubernetes. You need to:

- Create a cluster (in the us-east1-b zone) to host the service.

```bash
gcloud container clusters create nucleus-cluster1 --zone us-east1-b
```
```bash
gcloud container clusters get-credentials nucleus-cluster1
```
```bash
kubectl get namespaces
```

- Use the Docker container hello-app (`gcr.io/google-samples/hello-app:2.0`) as a placeholder; the team will replace the container with their own work later.

```bash
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:2.0
```

- Expose the app on port `App port number` .

```bash
kubectl expose deployment hello-server --type=LoadBalancer --port $PORT
```
```bash
kubectl get service
```

### Create an HTTP load balancer

You will serve the site via nginx web servers, but you want to ensure that the environment is fault-tolerant. Create an HTTP load balancer with a managed instance group of **2 nginx web servers**. Use the following code to configure the web servers; the team will replace this with their own configuration later.

```bash
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

```bash
gcloud compute regions list
```
```bash
gcloud config set compute/zone us-east1-b
```
```bash
gcloud compute zones list
```
```bash
gcloud config set compute/region us-east1
```

You need to:

- Create an instance template.

```bash
gcloud compute instance-templates create nucleus-instance-templates1 \
   --region= \
   --tags=allow-health-check \
   --machine-type=n1-standard-1 \
   --metadata-from-file startup-script=startup.sh \
```

- Create a managed instance group.

```bash
gcloud compute instance-groups managed create nucleus-instance-group1 \
    --template=nucleus-instance-templates1 \
    --size=2
```

- Create a firewall rule named as `Firewall rule` to allow traffic (80/tcp).

```bash

gcloud compute firewall-rules create $FIREWALL_RULE \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```

- Create a health check.

```bash
gcloud compute http-health-checks create nucleus-health-check --port 80
```

- Create a backend service, and attach the managed instance group with named port (http:80).
```bash
gcloud compute instance-groups managed \
          set-named-ports nucleus-instance-group1 \
          --named-ports http:80
```
```bash
gcloud compute backend-services create nucleus-backend-service1 \
  --protocol=HTTP \
  --port-name=http \
  --http-health-checks nucleus-health-check \
  --global
```
```bash
gcloud compute backend-services add-backend nucleus-backend-service1 \
  --instance-group=nucleus-instance-group1 \
  --global
```

- Create a URL map, and target the HTTP proxy to route requests to your URL map.

```bash
gcloud compute url-maps create nucleus-map-http --default-service nucleus-backend-service1
```
```bash
gcloud compute target-http-proxies create nucleus-http-proxy1 \
    --url-map nucleus-map-http
```

- Create a forwarding rule.

```bash
gcloud compute addresses create nucleus-ipv4-1 \
  --ip-version=IPV4 \
  --global
```
```bash
gcloud compute forwarding-rules create nucleus-http-content-rule \
    --address=nucleus-ipv4-1 \
    --global \
    --target-http-proxy=nucleus-http-proxy1 \
    --ports=80
```
