# Lab 4: Failure Injection

## Summary

This lab walks through using Linkerd to inject failures into an application.

### What Do You Need?

* A Kubernetes cluster 
* A machine with the Linkerd CLI installed

## Failure Injection

We already have the books app installed in our cluster and injected with the Linkerd proxy from lab 2.

1. Check the health of the application:

   ```bash
   linkerd stat deploy
   ```

2. One of services in this application has been configured with an error rate. The purpose of this demo is to show that we can inject errors without needing any support for this in the application. Remove that configured error rate:

   ```bash
   kubectl edit deploy/authors
   # Find and remove these lines:
   #        - name: FAILURE_RATE
   #          value: "0.5"
   ```

3. Check the health of the application again:

   ```text
   linkerd stat deploy
   ```

   We should now see the application is healthy and has a 100% success rate.

4. Now we can create our error service. Here I will use NGINX configured to respond only with HTTP status code 500. Create a file called `error-injector.yaml`

   ```text
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: error-injector
     labels:
       app: error-injector
   spec:
     selector:
       matchLabels:
         app: error-injector
     replicas: 1
     template:
       metadata:
         labels:
           app: error-injector
       spec:
         containers:
           - name: nginx
             image: nginx:alpine
             ports:
             - containerPort: 80
               name: nginx
               protocol: TCP
             volumeMounts:
               - name: nginx-config
                 mountPath: /etc/nginx/nginx.conf
                 subPath: nginx.conf
         volumes:
           - name: nginx-config
             configMap:
               name: error-injector-config
   ---
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app: error-injector
     name: error-injector
   spec:
     clusterIP: None
     ports:
     - name: service
       port: 7002
       protocol: TCP
       targetPort: nginx
     selector:
       app: error-injector
     type: ClusterIP
   ---
   apiVersion: v1
   data:
    nginx.conf: |2

       events {
           worker_connections  1024;
       }

       http {
           server {
               location / {
                   return 500;
               }
           }
       }
   kind: ConfigMap
   metadata:
     name: error-injector-config
   ```

5. Deploy the manifest:

   ```text
   kubectl apply -f error-injector.yaml
   ```

6. Now we can create a traffic split resource which will direct 10% of the books service to the error service. Create a file called `error-split.yaml`:

   ```text
   apiVersion: split.smi-spec.io/v1alpha1
   kind: TrafficSplit
   metadata:
     name: error-split
   spec:
     service: books
     backends:
     - service: books
       weight: 900m
     - service: error-injector
       weight: 100m
   ```

7. Deploy the manifest:

   ```text
   kubectl apply -f error-split.yaml
   ```

8. View the error rate:

   ```text
   linkerd routes deploy/webapp --to service/books
   ```

9. Explore how the application handles these errors:

   ```bash
   kubectl port-forward deploy/webapp 7000 & 
   open http://localhost:7000
   ```

   If you refresh the page a few times, you will sometimes see an internal server error page.

10. Restore the health of the application by deleting the traffic split resource:

    ```text
    kubectl delete trafficsplit/error-split
    ```

11. To remove the books app and the `booksapp` namespace from your cluster, run:

    ```text
    curl -sL https://run.linkerd.io/booksapp.yml \
      | kubectl -n booksapp delete -f - \
      && kubectl delete ns booksapp
    ```

## Additional Reading

[Main Linkerd 2 GitHub Repo](https://github.com/linkerd/linkerd2)

[Linkerd Website](https://github.com/linkerd/website)

