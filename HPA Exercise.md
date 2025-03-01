# **Horizontal Pod Autoscaling (HPA) with Nginx on Rancher Desktop**

## **Overview**

### We’ll do the following:

1. **Check cluster context** to ensure commands run on Rancher Desktop.
2. **Deploy Nginx** with CPU requests/limits so autoscaling can measure usage.
3. **Create a NodePort Service** to expose Nginx.
4. **Configure an HPA** that scales between 1 and 5 pods when CPU usage exceeds 50%.
5. **Load-test** using hey to generate CPU usage and observe scaling.
6. **Clean up** after we’re done.

## **Prerequisites**
1. **Rancher Desktop** installed and running, with Kubernetes enabled.
2. **kubectl** installed and configured.
3. **Metrics Server** enabled (often Rancher Desktop includes this by default, but if not, you may need to install or enable it).
4. [**hey**](github.com/rakyll/hey)(load-testing tool).


## **1. Confirm You’re on the Rancher Desktop Cluster**

Before applying any manifests, ensure you’re using the Rancher Desktop cluster context:

```
kubectl config get-contexts
```

![ImageAlt](https://github.com/shikharsaxena123/K8s/blob/8fce5ed53f16735f73bd5a5c5b63bb1992395bae/get-context.png)

If rancher-desktop is not the one marked with *, switch to it:

```
kubectl config use-context rancher-desktop
```

_`kubectl config use-context <CONTEXT_NAME>`: Tells kubectl to run commands against the specified context (cluster)._

## **2. Create the Nginx Deployment**

Below is a sample nginx-deployment.yaml manifest. This defines an Nginx Deployment with CPU requests (100m) and CPU limits (300m). 
The HPA will only work if CPU requests are set, because autoscaling calculates usage relative to the requested CPU.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 300m
```

_`kind`: Deployment: Manages a ReplicaSet which in turn manages Pods._

_`replicas`: `1`: Start with `1` replica (pod)._

_`matchLabels`: app: nginx: Selector that ties this Deployment to pods with label `app=nginx`._

_`image`: nginx:latest: Nginx container image._

_`containerPort`: `80`: The container is listening on port `80`._

_`resources.requests.cpu`: `100m`: Requests `0.1` CPU. The HPA uses this as the denominator for CPU utilization calculations._

_`resources.limits.cpu`: `300m`: The maximum CPU the container can use is `0.3` CPU cores._

### Apply the manifest

```
kubectl apply -f nginx-deployment.yaml
```

_`kubectl apply -f file.yaml`: Creates or updates the Kubernetes resources defined in file.yaml._

## **3. Create a NodePort Service**

We need a way to access Nginx from outside the cluster. A NodePort service exposes port 80 on a randomly allocated high port (like 30000+). Here’s nginx-service.yaml:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    # nodePort: 32266 (optional, you can specify or let Kubernetes assign automatically)
```

_`kind`: Service with type: `NodePort`: Exposes a port on each node’s IP address._

_`selector`: app: nginx: Routes traffic to any pod with app=nginx._

_`port`: `80` and targetPort: `80`: The Service port is `80`, which forwards to container port `80`._

_`nodePort`: 32266 (optional): If you don’t specify it, Kubernetes will assign a random NodePort. If you prefer a specific one, uncomment and set it._


### Apply the service:

```
kubectl apply -f nginx-service.yaml
```

Now, check your service:

```
kubectl get svc
```

Look for nginx-service, which might show a NodePort of, say, 80:32266/TCP.

_`kubectl get svc`: Lists all services in the current namespace (default). The NodePort column indicates which port is assigned on the node._

## **4. Create the Horizontal Pod Autoscaler**

We’ll define an HPA that maintains 50% average CPU usage across pods. Here’s nginx-hpa.yaml:

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

_`scaleTargetRef` points to your Deployment named nginx-deployment._

_`minReplicas`: 1 and `maxReplicas`: 5: The HPA will never go below 1 pod or above 5 pods._

_`averageUtilization`: 50: If the average CPU usage across pods exceeds 50% of their requested CPU, the HPA scales up. If it’s below 50%, it may scale down._

### Apply the HPA:

```
kubectl apply -f nginx-hpa.yaml
```

### Check status:

```
kubectl get hpa
```

_`kubectl get hpa`: Lists Horizontal Pod Autoscalers. Shows the current CPU usage vs. target, current replicas, desired replicas, and more._

## **5. Verify Everything Is Up**

```
kubectl get pods
```

You should see nginx-deployment-xxxxxx in a Running state.

### Test With curl

Find the NodePort and use that for testing.

### Try curling:

You should see the Nginx welcome page HTML.

## **6. Generate Load and Observe Autoscaling**

Using 'hey'
To test autoscaling, we’ll stress CPU usage by making many requests:

```
hey -z 1m -q 50 http://127.0.0.1:32266
```

_-z 1m: Run for 1 minute.
-q 50: Send 50 requests per second._


_If 50 QPS doesn’t raise CPU usage enough, increase to 100, 200, etc.:_

```
hey -z 1m -q 200 http://127.0.0.1:32266
```

_`hey`: A small load generator that floods the provided URL with HTTP requests._

### Watch the HPA

```
kubectl get hpa -w
```

Alternatively, watch your pods:

```
kubectl get pods -l app=nginx -w
```

You’ll see new pods start if the load triggers scaling.

## **Cleanup**

```
kubectl delete -f nginx-hpa.yaml
kubectl delete -f nginx-service.yaml
kubectl delete -f nginx-deployment.yaml
```

_`kubectl delete -f file.yaml`: Deletes the resources defined in the YAML (HPA, Service, Deployment)._

## **Recap**

1. We deployed Nginx with CPU requests so HPA can measure usage.
2. We exposed it via a NodePort Service and verified it with curl.
3. We created an HPA that watches average CPU usage.
4. We load-tested using hey, increasing QPS if needed to trigger scaling.
5. We observed new pods come online under higher CPU load.
6. This completes a simple demonstration of how Horizontal Pod Autoscaling works in Kubernetes using Nginx and Rancher Desktop.
