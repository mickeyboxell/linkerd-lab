# Lab 3: Canary Deployments

## Summary

This lab walks through the process of installing and configuring Flagger, a progressive delivery tool. Flagger and Linkerd will be used to perform a canary deployment.

### What Do You Need?

* A Kubernetes cluster with Linkerd installed
* A machine with the Linkerd CLI installed

## Install and Configure Flagger

1. Ensure your `kubectl` is version 1.14 or greater or install Kustomize \([https://github.com/kubernetes-sigs/kustomize](https://github.com/kubernetes-sigs/kustomize)\)
2. Install Flagger in the `linkerd` namespace:

   ```text
   kubectl apply -k github.com/weaveworks/flagger//kustomize/linkerd
   ```

3. Create a `test` namespace and enable Linkerd proxy injection:

   ```text
   kubectl create ns test
   kubectl annotate namespace test linkerd.io/inject=enabled
   ```

4. Install the load testing service to generate traffic during the canary analysis:

   ```text
   kubectl apply -k github.com/weaveworks/flagger//kustomize/tester
   ```

5. Create a deployment and a horizontal pod autoscaler:

   ```text
   kubectl apply -k github.com/weaveworks/flagger//kustomize/podinfo
   ```

6. Create a canary custom resource for the `podinfo` deployment. Create `podinfo-canary.yaml` and add the following:

   ```text
   apiVersion: flagger.app/v1alpha3
   kind: Canary
   metadata:
     name: podinfo
     namespace: test
   spec:
     # deployment reference
     targetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: podinfo
     # HPA reference (optional)
     autoscalerRef:
       apiVersion: autoscaling/v2beta1
       kind: HorizontalPodAutoscaler
       name: podinfo
     # the maximum time in seconds for the canary deployment
     # to make progress before it is rollback (default 600s)
     progressDeadlineSeconds: 60
     service:
       # container port
       port: 9898
     canaryAnalysis:
       # schedule interval (default 60s)
       interval: 30s
       # max number of failed metric checks before rollback
       threshold: 5
       # max traffic percentage routed to canary
       # percentage (0-100)
       maxWeight: 50
       # canary increment step
       # percentage (0-100)
       stepWeight: 5
       # Linkerd Prometheus checks
       metrics:
       - name: request-success-rate
         # minimum req success rate (non 5xx responses)
         # percentage (0-100)
         threshold: 99
         interval: 1m
       - name: request-duration
         # maximum req duration P99
         # milliseconds
         threshold: 500
         interval: 30s
       # testing (optional)
       webhooks:
         - name: acceptance-test
           type: pre-rollout
           url: http://flagger-loadtester.test/
           timeout: 30s
           metadata:
             type: bash
             cmd: "curl -sd 'test' http://podinfo-canary:9898/token | grep token"
         - name: load-test
           type: rollout
           url: http://flagger-loadtester.test/
           metadata:
             cmd: "hey -z 2m -q 10 -c 2 http://podinfo:9898/"
   ```

7. Apply the `podinfo-canary.yaml` to the cluster:

   ```text
   kubectl apply -f ./podinfo-canary.yaml
   ```

   When the canary analysis starts, Flagger will call the pre-rollout webhooks before routing traffic to the canary. The canary analysis will run for five minutes while validating the HTTP metrics and rollout hooks every half a minute. Flagger implements a control loop that gradually shifts traffic to the canary while measuring key performance indicators like HTTP requests success rate, requests average duration and pod health. Based on analysis of the KPIs a canary is promoted or aborted. After a couple of seconds Flagger will create the canary objects.

   After five minutes the `podinfo` deployment will be scaled to zero and the traffic to `podinfo.test` will be routed to the primary pods. During the canary analysis, the `podinfo-canary.test` address can be used to directly target the canary pods.

## Trigger a Canary Deployment

Canary deployments can be triggered by changes in any of the following objects:

* Deployment PodSpec \(container image, command, ports, env, resources, etc\)
* ConfigMaps mounted as volumes or mapped to environment variables
* Secrets mounted as volumes or mapped to environment variables
* Trigger a canary deployment by updating the pod image:

  ```text
  kubectl -n test set image deployment/podinfo \
  podinfod=quay.io/stefanprodan/podinfo:1.7.1
  ```

* Flagger detects that the deployment revision changed and starts a new rollout. View the rollout by describing the pod:

  ```text
  kubectl -n test describe canary/podinfo
  ```

* Monitor all canaries with:

  ```text
  watch kubectl get canaries --all-namespaces
  ```

## Automated Rollback

During the canary analysis you can generate HTTP 500 errors and high latency to test if Flagger pauses and rolls back the faulted version.

1. Trigger another canary deployment:

   ```text
   kubectl -n test set image deployment/podinfo \
   podinfod=quay.io/stefanprodan/podinfo:1.7.2
   ```

2. Exec into the `loadtester` pod:

   ```text
   kubectl -n test exec -it flagger-loadtester-xx-xx sh
   ```

3. Generate HTTP 500 errors:

   ```text
   watch -n 1 curl http://podinfo-canary.test:9898/status/500
   ```

4. In another shell generate latency:

   ```text
   watch -n 1 curl http://podinfo-canary.test:9898/delay/1
   ```

5. When the number of failed checks reaches the canary analysis threshold, the traffic is routed back to the primary, the canary is scaled to zero and the rollout is marked as failed. View the status of the rollout with:

   ```text
   kubectl -n test describe canary/podinfo
   ```

## Custom Metrics

The canary analysis can be extended with Prometheus queries.

1. We can define a check for HTTP 404 not found errors. Edit the `canaryAnalysis` of the `podinfo` deployment:

   ```text
   kubectl edit deployment/podinfo
   ```

   Add the following metric:

   ```text
     canaryAnalysis:
       metrics:
       - name: "404s percentage"
         threshold: 3
         query: |
           100 - sum(
               rate(
                   response_total{
                       namespace="test",
                       deployment="podinfo",
                       status_code!="404",
                       direction="inbound"
                   }[1m]
               )
           )
           /
           sum(
               rate(
                   response_total{
                       namespace="test",
                       deployment="podinfo",
                       direction="inbound"
                   }[1m]
               )
           )
           * 100
   ```

   This will validate the canary version by checking if the HTTP 404 request per second percentage is below three percent of total traffic. If the 404's rate reaches the 3% threshold, then the analysis will be aborted and the canary is marked as failed.

2. Trigger a canary deployment by updating the container image:

   ```text
   kubectl -n test set image deployment/podinfo \
   podinfod=quay.io/stefanprodan/podinfo:1.7.3
   ```

3. In another shell generate 404's:

   ```text
   watch -n 1 curl http://podinfo-canary:9898/status/404
   ```

4. Watch the Flagger logs to see the canary failure and rollback:

   ```text
   kubectl -n linkerd logs deployment/flagger -f | jq .msg
   ```

## Additional Reading

[Main Linkerd 2 GitHub Repo](https://github.com/linkerd/linkerd2)

[Linkerd Website](https://github.com/linkerd/website)

[Flagger](https://docs.flagger.app/>)

