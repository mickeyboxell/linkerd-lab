# Lab 5: Debugging Your Service

## Summary

This lab walks through using Linkerd and the Linkerd debug container to debug a sample application.

### What Do You Need?

* A Kubernetes cluster 
* A machine onto which you can install the Linkerd CLI

## Install the Sample Application

To get a feel for how Linkerd would work for one of your services, you can install a demo application. The emojivoto application is a standalone Kubernetes application that uses a mix of gRPC and HTTP calls to allow the users to vote on their favorite emojis.

1. Install the emojivoto application into the `emojivoto` namespace by running:

   ```text
   curl -sL https://run.linkerd.io/emojivoto.yml \
     | kubectl apply -f -
   ```

2. Before we mesh it, let’s take a look at the app. If you’re using [Docker Desktop](https://www.docker.com/products/docker-desktop) at this point you can visit [http://localhost](http://localhost/) directly. If you’re not using Docker Desktop, we’ll need to forward the `web-svc` service. To forward `web-svc` locally to port 8080, you can run:

   ```text
   kubectl -n emojivoto port-forward svc/web-svc 8080:80
   ```

3. Now visit [http://localhost:8080](http://localhost:8080/). Voila! The emojivoto app in all its glory.

   Clicking around, you might notice that some parts of the emojivoto application are broken. If you click on a doughnut emoji, you will get a 404 page. These errors are intentional and we can use Linkerd to identify the problem.

4. Add Linkerd to emojivoto by running:

   ```text
   kubectl get -n emojivoto deploy -o yaml \
     | linkerd inject - \
     | kubectl apply -f -
   ```

   This command retrieves all of the deployments running in the `emojivoto` namespace, runs the manifest through `linkerd inject`, and then reapplies it to the cluster. The `linkerd inject`c ommand adds annotations to the pod spec instructing Linkerd to add the data plane’s proxy to the pod spec as a container.

5. Verify that everything worked the way it should with the data plane with:

   ```text
   linkerd -n emojivoto check --proxy
   ```

## The Debug Container

Linkerd provides a **debug sidecar** in cases where you need network-level visibility into packets entering and leaving your application. Similar to how [proxy sidecar injection](https://linkerd.io/2/features/proxy-injection/) works, you add a debug sidecar to a pod by setting the `config.linkerd.io/enable-debug-sidecar: true` annotation at pod creation time. For convenience, the `linkerd inject` command provides an `--enable-debug-sidecar` option that does this annotation for you.

The debug sidecar image contains [`tshark`](https://www.wireshark.org/docs/man-pages/tshark.html), `tcpdump`, `lsof`, and `iproute2`. Once installed, it starts automatically logging all incoming and outgoing traffic with `tshark`, which can then be viewed with `kubectl logs`. Alternatively, you can use `kubectl exec` to access the container and run commands directly.

1. Debug traffic to all pods in the `voting` service of the emojivoto application with:

   ```bash
   kubectl -n emojivoto get deploy/voting -o yaml \
     | linkerd inject --enable-debug-sidecar - \
     | kubectl apply -f -
   ```

2. Confirm that the debug container is running by listing all the containers in pods with the `voting-svc` label:

   ```text
   kubectl get pods -n emojivoto -l app=voting-svc \
     -o jsonpath='{.items[*].spec.containers[*].name}'
   ```

3. Watch live tshark output from the logs by simply running:

   ```bash
   kubectl -n emojivoto logs deploy/voting linkerd-debug -f
   ```

4. You can exec to the container and run your own commands in the context of the network. For example, if you want to inspect the HTTP headers of the requests, you could run something like this:

   ```bash
   user@local$ kubectl -n emojivoto exec -it voting-7cf4784dd8-qxjv4 \
     -c linkerd-debug -- /bin/bash
   root@voting-7cf4784dd8-qxjv4:/#
   root@voting-7cf4784dd8-qxjv4:/# tshark -i any -f "tcp" -V -Y "http.request"
   Running as user "root" and group "root". This could be dangerous.
   Capturing on 'any'

   ...
   ```

   Of course, this only works if you have the ability to `exec` into arbitrary containers in the Kubernetes cluster. [`linkerd tap`](https://linkerd.io/2/reference/cli/tap/) provides an alternative to this approach.

## Additional Reading

[Main Linkerd 2 GitHub Repo](https://github.com/linkerd/linkerd2)

[Linkerd Website](https://github.com/linkerd/website)

