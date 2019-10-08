# Service Mesh: Beyond the Hype with Linkerd

![Linkerd](https://user-images.githubusercontent.com/9226/33582867-3e646e02-d90c-11e7-85a2-2e238737e859.png)



# Table of Contents

- Lab 0: Oracle Cloud Account and Kubernetes Cluster
- Lab 1: Install Linkerd
- Lab 2: Demo Application
- Lab 3: Canary Deployments
- Lab 4: Failure Injection
- Lab 5: Debugging Your Service



# Lab 0: Oracle Cloud Account and Kubernetes Cluster 

## Oracle Cloud Account 

1. Click on the following link to create your [Free Account](https://myservices.us.oraclecloud.com/mycloud/signup?language=en&intcmp=NAMK180429P00004:ow:evp:cpo::RC_NAMK180826P00001:OKE_HOL), and complete all the required steps to get your free Oracle Cloud Trial Account. When you complete the registration process you'll receive a credit that will enable you to complete the lab for free. Additionally, you'll have time left over to continue to explore the Oracle Cloud. Soon after requesting your trial you will receive the following email from oraclecloudadmin_ww@oracle.com: 

![img](https://oracle.github.io/learning-library/workshops/container-native-development-with-oke/images/oraclecode/code_9.png)



## Kubernetes Cluster Creation 

1. Within the root compartment of your tenancy, a policy statement (`Allow service OKE to manage all-resources in tenancy`) must be defined to give Container Engine for Kubernetes access to resources in the tenancy. See [Create Required Policy for Container Engine for Kubernetes](https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengpolicyconfig.htm#PolicyPrerequisitesService). 

2. Using default settings to create a 'quick cluster' with new network resources as required. This approach is the fastest way to create a new cluster. If you accept all the default values, you can create a new cluster in just a few clicks. New network resources for the cluster are created automatically, along with a node pool and worker nodes. Note that worker nodes in a 'quick cluster' are created in private subnets, so a NAT gateway is also created (in addition to an internet gateway). 

3. In the Console, open the navigation menu. Under **Solutions and Platform**, go to **Developer Services**and click **Container Clusters**.

4. Choose a **Compartment** you have permission to work in, and in which you want to create both the new cluster and the associated network resources.

5. On the **Cluster List** page, click **Create Cluster**.

6. Either just accept the default configuration details for the new cluster, or specify alternatives as follows: 

   - **Name:** The name of the new cluster. Either accept the default name or enter a name of your choice. Avoid entering confidential information.**Kubernetes**
   - **Version:** The version of Kubernetes to run on the master nodes and worker nodes of the cluster. Either accept the default version or select a version of your choice. Amongst other things, the Kubernetes version you select determines the default set of admission controllers that are turned on in the created cluster (the set follows the recommendation given in the [Kubernetes documentation](https://kubernetes.io/docs/admin/admission-controllers/#is-there-a-recommended-set-of-admission-controllers-to-use) for that version).

7. Select **Quick Create** to create a new cluster with default settings, along with new network resources for the new cluster. The **Create Virtual Cloud Network** panel shows the network resources that will be created for you by default. The **Create Node Pool** panel shows the fixed properties of the first node pool in the cluster that will be created for you: 

   - The name of the node pool (always pool1)
   - The compartment in which the node pool will be created (always the same as the one in which the new network resources will reside)
   - The version of Kubernetes that will run on each worker node in the node pool (always the same as the version specified for the master nodes)the image to use on each node in the node pool

8. The **Create Node Pool** panel also contains some node pool properties that you can change, but which have been given sensible defaults. Either just accept all the default configuration details and skip ahead to the next step to create the cluster immediately, or specify alternatives as follows: Either accept the default configuration details for the node pool, or specify alternatives in the Create Node Pool panel as follows: 

   - **Shape:** The shape to use for each node in the node pool. The shape determines the number of CPUs and the amount of memory allocated to each node. The list shows only those shapes available in your tenancy that are supported by Container Engine for Kubernetes.
   - **Quantity per Subnet:** The number of worker nodes to create for the node pool in each private subnet.
   - **Public SSH Key:** (Optional) The public key portion of the key pair you want to use for SSH access to each node in the node pool. The public key is installed on all worker nodes in the cluster. Note that if you don't specify a public SSH key, Container Engine for Kubernetes will provide one. However, since you won't have the corresponding private key, you will not have SSH access to the worker nodes. Note that because worker nodes in a 'quick cluster' are in private subnets, you cannot use SSH to access them directly (see [Connecting to Worker Nodes in Private Subnets Using SSH](https://docs.cloud.oracle.com/iaas/Content/ContEng/Tasks/contengconnectingworkernodesusingssh.htm#Connecti2)).
   - **Kubernetes Labels:** One or more labels (in addition to a default label) to add to worker nodes in the node pool to enable the targeting of workloads at specific node pools.

9. Either accept the defaults for the remaining cluster details, or specify alternatives in the **Additional Add Ons** panel as follows: 

   - **Kubernetes Dashboard Enabled:** Select if you want to use the Kubernetes Dashboard to deploy and troubleshoot containerized applications, and to manage Kubernetes resources. See [Starting the Kubernetes Dashboard](https://docs.cloud.oracle.com/iaas/Content/ContEng/Tasks/contengstartingk8sdashboard.htm).

   - **Tiller (Helm) Enabled:** Select if you want Tiller (the server portion of Helm) to run in the Kubernetes cluster. With Tiller running in the cluster, you can use Helm to manage Kubernetes resources.

10. Click **Create** to create the new network resources and the new cluster. Container Engine for Kubernetes starts creating the following resources:

    - the network resources (such as the VCN, internet gateway, NAT gateway, route tables, security lists, private subnets), named oke-<resource-type>-quick-<cluster-name>-<creation-date>

    - the cluster, with the name you specified
    - the node pool, named pool1
    - worker nodes, with names in the format `oke-c<part-of-cluster-OCID>-n<part-of-node-pool-OCID>-s<part-of-subnet-OCID>-<slot>`

    Note that if the cluster is not created successfully for some reason (for example, if you have insufficient permissions or if you've exceeded the cluster limit for the tenancy), any network resources created during the cluster creation process are not deleted automatically. You will have to manually delete any such unused network resources. Click **Close** to return to the Console.

11. Get the tenancy and user OCID's from the Console, which is located at [https://console.us-ashburn-1.oraclecloud.com](https://console.us-phoenix-1.oraclecloud.com/). Open the navigation menu, under Governance and Administration, go to **Administration** and click **Tenancy Details**. The tenancy OCID is shown under **Tenancy Information**. Click **Copy** to copy it to your clipboard. Get the user's OCID in the Console on the page showing the user's details. To get to that page: If you're signed in as the user: Open the User menu (User menu icon) and click User Settings.

12. Install the OCI CLI. To run the installer script, run the following command: 

    ```
    bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"
    ```

13. To have the CLI walk you through the first-time setup process, use:

    ```
    oci setup config
    ```

    The command prompts you for the information required for the config file and the API public/private keys. The setup dialog generates an API key pair and creates the config file. For more information refer to [Installing the CLI](<https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/cliinstall.htm>). 

14. Download kubeconfig. In the Console, open the navigation menu. Under Solutions and Platform, go to Developer Services and click Container Clusters. On the Cluster List page, click the name of the cluster you want to access using kubectl and the Kubernetes Dashboard. The Cluster page shows details of the cluster. Click the Access Kubeconfig button to display the How to Access Kubeconfig dialog box.Create a directory to contain the kubeconfig file. By default, the expected directory name is `$HOME/.kube`. For example, on Linux, enter the following command (or copy and paste it from the **How to Access Kubeconfig** dialog box):

    ```
    mkdir -p $HOME/.kube
    ```

15. Run the Oracle Cloud Infrastructure CLI command to download the kubeconfig file and save it in a location accessible to kubectl and the Kubernetes Dashboard. For example, on Linux, enter the following command (or copy and paste it from the **How to Access Kubeconfig** dialog box) where ocid1.cluster.oc1.phx.aaaaaaaaae... is the OCID of the current cluster. For convenience, the command in the **How to Access Kubeconfig** dialog box already includes the cluster's OCID: 

    ```
    oci ce cluster create-kubeconfig --cluster-id ocid1.cluster.oc1.phx.aaaaaaaaae... --file $HOME/.kube/config  --region us-phoenix-1
    ```

    Note that if a kubeconfig file already exists in the location you specify, details about the cluster will be added as a new context to the existing kubeconfig file. The `current-context:` element in the kubeconfig file will be set to point to the newly added context. 

16. Set the value of the KUBECONFIG environment variable to point to the name and location of the kubeconfig file. For example, on Linux, enter the following command (or copy and paste it from the **How to Access Kubeconfig** dialog box):

    ```
    export KUBECONFIG=$HOME/.kube/config
    ```

17. Test your kubeconfig and `kubectl` are configured correctly with:

    ```
    kubectl version
    ```

    Verify that kubectl can connect to the cluster with: 

    ```
    kubectl get nodes
    ```



# Lab 1: Install Linkerd



1. Download the Linkerd CLI:

   ```
   curl -sL https://run.linkerd.io/install | sh
   ```

2. Add Linkerd to your path:

   ```
   export PATH=$PATH:$HOME/.linkerd2/bin
   ```

3. Verify the CLI was properly installed with: 

   ```
   linkerd version
   ```

4. Validate your cluster is configured correct for Linkerd with:

   ```
   linkerd check --pre
   ```

5. Generate a manifest file and apply it to your cluster with:

   ```
   linkerd install | kubectl apply -f -
   ```

   In addition to creating the `linkerd` namespace, this command installs the following resources onto your Kubernetes cluster:

   - ClusterRole
   - ClusterRoleBinding
   - CustomResourceDefinition
   - MutatingWebhookConfiguration
   - PodSecurityPolicy
   - Role
   - RoleBinding
   - Secret
   - ServiceAccount
   - ValidatingWebhookConfiguration

6. Verify your install with:

   ```
   linkerd check
   ```

7. See what components were installed: 

   ```
   kubectl -n linkerd get deploy
   ```

8. Navigate to the dashboard with:

   ```
   linkerd dashboard
   ```


# Lab 2: Demo Application 



## Install the Demo Application 

1. Install the books app to your cluster with:

   ```
   kubectl create ns booksapp && \
     curl -sL https://run.linkerd.io/booksapp.yml \
     | kubectl -n booksapp apply -f -
   ```

   This command creates a `booksapp` namespace for the demo application, downloads its Kubernetes resource manifest, and uses `kubectl` to apply it to your cluster.

2. Verify the installation with:

   ```
   kubectl -n booksapp get all
   ```

   You should see deployments for `authors`, `books`, `traffic`, and `webapp`. 

3. Access the application via portforward by running:

   ```
   kubectl -n booksapp port-forward svc/webapp 7000 &
   ```

    Navigate to http://localhost:7000/

   <Image>



## Add Linkerd to the Application 

The application contains an error that results in the **Add Book** link failing 50% of the time with an internal server error. We can use Linkerd to diagnose the root cause. 

1. Add the Linkerd sidecar to the demo application with:

   ```bash
   kubectl get -n booksapp deploy -o yaml \
     | linkerd inject - \
     | kubectl apply -f -
   ```

   This will use the `linkerd inject` annotate the resources in the namespace to specify they should have the Linkerd proxy added and then applies the change with `kubectl apply`. This is a rolling process that will not cause your application to incur downtime. 

   Annotations and Linkerd proxies can be removed from the application with `linkerd uninject`. 



## Debug the Error 

1. Open the Linkerd dashboard:

   ```
   linkerd dashboard &
   ```

2. The **Overview** tab will show all of the services running in the `booksapp` namespace. There will be success rate, requests per second, and latency percentiles. Notice that the success rate for `webapp` is not 100%. This is because the traffic generator is submitting new books. You can do the same thing yourself and push that success rate even lower. Click on `webapp` in the Linkerd dashboard for a live debugging session.

   ![overview](/Users/mboxell/Desktop/Desktop/overview.png)

3. On the **Detail** view for the `webapp` service you’ll see that `webapp` is taking traffic from `traffic` (the load generator), and it has two outgoing dependencies: `authors` and `book`. One is the service for pulling in author information and the other is the service for pulling in book information. We can see that while the `authors` service has a 100% success rate the `books` service is less than 100%. A failure in the dependent `books` service may be the cause of the errors that `webapp` is returning. 

   ![detail](/Users/mboxell/Desktop/detail.png)

4. Scroll down the page to see the live list of all traffic endpoints that `webapp` is receiving. The information on this page is populated based on the `linkerd top` command, which is used to display sorted information about live traffic. 

   ![top](/Users/mboxell/Desktop/Desktop/top.png)

5. Inbound traffic coming from the `webapp` service going to the `books` service is failing a significant percentage of the time. This may explain why `webapp` is throwing intermittent failures. The information on this page is populated based on the `linkerd tap` command, which is used to listen to a traffic stream. Click on the 🔬 icon to look at the request and response stream.

   ![tap](/Users/mboxell/Desktop/Desktop/tap.png)

6. Many of these requests are returning HTTP 500 errors. Click on the **∨** in the left most column of any request to see additional details.

   ![tap_detail](/Users/mboxell/Desktop/Desktop/tap_detail.png)

   This shows how to diagnose an intermittent issue that affecting a single route. You now have everything you need to open a bug report explaining exactly what the root cause is. If the `books` service was your own, you know exactly where to look in the code. 



## Service Profiles

[Service profiles](https://linkerd.io/2/features/service-profiles/) define the routes that you’re serving and, among other things, allow for the collection of metrics on a per route basis. The information is stored in Prometheus and can be queried at a later time. 

1. One of the easiest ways to get service profiles setup is by using existing [OpenAPI (Swagger)](https://swagger.io/docs/specification/about/) specs. This demo has published specs for each of its services. You can create a service profile for `webapp` by running: 

   ```bash
   curl -sL https://run.linkerd.io/booksapp/webapp.swagger \
     | linkerd -n booksapp profile --open-api - webapp \
     | kubectl -n booksapp apply -f -
   ```

   This command will fetch the swagger specification for `webapp`, convert the spec into a service profile by means of the `profile` command, and apply the configuration to the cluster. Similar to `inject`, the `profile` command works by creating annotations. Here you can see the profile created for the `webapp` service in the command above: 

   ```yaml
   apiVersion: linkerd.io/v1alpha1
   kind: ServiceProfile
   metadata:
     creationTimestamp: null
     name: webapp.booksapp.svc.cluster.local
     namespace: booksapp
   spec:
     routes:
     - condition:
         method: GET
         pathRegex: /
       name: GET /
     - condition:
         method: POST
         pathRegex: /authors
       name: POST /authors
     - condition:
         method: GET
         pathRegex: /authors/[^/]*
       name: GET /authors/{id}
     - condition:
         method: POST
         pathRegex: /authors/[^/]*/delete
       name: POST /authors/{id}/delete
     - condition:
         method: POST
         pathRegex: /authors/[^/]*/edit
       name: POST /authors/{id}/edit
     - condition:
         method: POST
         pathRegex: /books
       name: POST /books
     - condition:
         method: GET
         pathRegex: /books/[^/]*
       name: GET /books/{id}
     - condition:
         method: POST
         pathRegex: /books/[^/]*/delete
       name: POST /books/{id}/delete
     - condition:
         method: POST
         pathRegex: /books/[^/]*/edit
       name: POST /books/{id}/edit
   ```

   `name` refers to the FQDN of your Kubernetes service, in this case it is `webapp.booksapp.svc.cluster.local`. Linkerd uses the `Host` header of requests to associate service profiles with requests. When the proxy sees a `Host` header of `webapp.booksapp.svc.cluster.local`, it will use that to look up the service profile’s configuration.

   Routes are simple conditions that contain a method (`GET` for example) and a regex to match the path. This allows you to group REST style resources together instead of seeing a huge list. The names for routes can be whatever you’d like. For this demo, the method is appended to the route regex.

2. Get profiles for `authors` and `books` with the following command: 

3. ```bash
   curl -sL https://run.linkerd.io/booksapp/authors.swagger \
     | linkerd -n booksapp profile --open-api - authors \
     | kubectl -n booksapp apply -f -
   curl -sL https://run.linkerd.io/booksapp/books.swagger \
     | linkerd -n booksapp profile --open-api - books \
     | kubectl -n booksapp apply -f -
   ```

4. As seen in the debugging section, the `tap` feature of Linkerd providers a live view of the `:authority` or `Host` header as well as the `:path` and `rt_route` used for every request. To view the live requests for `webapp`, run: 

   ```
   linkerd -n booksapp tap deploy/webapp -o wide | grep req
   ```

   These metrics come from the [`linkerd route`](https://linkerd.io/2/reference/cli/routes/) command, which is used to display route stats. A similar command,  [`linkerd stat`](https://linkerd.io/2/reference/cli/stat/) is used to display traffic stats about one or many resources. To see the metrics that have accumulated so far, run: 

   ```
   linkerd -n booksapp routes svc/webapp
   ```

   This will output a table of all the routes observed. The `[DEFAULT]` route is a catch all for anything that does not match the service profile.

5. Profiles can also be used to observe *outgoing* requests as well as *incoming* requests. To do that, run:

   ```bash
   linkerd -n booksapp routes deploy/webapp --to svc/books
   ```

   This will show all requests and routes that originate in the `webapp` deployment and are destined to the `books` service. Similarly to using `tap` and `top` views in the [debugging](https://linkerd.io/2/tasks/books/#debugging) section, the root cause of errors in this demo is immediately apparent. 



## Retries

Rather than updating the code, we can use Linkerd to retry requests to the failing endpoint. This will increase request latency, as requests will be retried multiple times, but not require rolling out a new version. 

1. In this application, the success rate of requests from the `books` deployment to the `authors`service is poor. To see these metrics, run:

   ```bash
   linkerd -n booksapp routes deploy/books --to svc/authors
   ```

   We can see requests from books to authors are failing about 50% of the time. 

2. To correct this, let’s edit the authors service profile and make those requests retryable by running:

   ```
   kubectl -n booksapp edit sp/authors.booksapp.svc.cluster.local
   ```

   And adding `isRetryable` to the troubled `HEAD /authors/{id}.json` route in the deployment spec: 

   ```
   spec:
     routes:
     - condition:
         method: HEAD
         pathRegex: /authors/[^/]*\.json
       name: HEAD /authors/{id}.json
       isRetryable: true ### ADD THIS LINE ###
   ```

3. After editing the service profile, Linkerd will begin to retry requests to this route automatically. We see a nearly immediate improvement in success rate by running:

   ```bash
   linkerd -n booksapp routes deploy/books --to svc/authors -o wide
   ```

   You’ll notice that the `-o wide` flag has added some columns to the `routes` view. These show the difference between `EFFECTIVE_SUCCESS` and `ACTUAL_SUCCESS`. The difference between these two show how well retries are working. `EFFECTIVE_RPS` and `ACTUAL_RPS` show how many requests are being sent to the destination service and and how many are being received by the client’s Linkerd proxy. With retries automatically happening now, success rate looks great but the p95 and p99 latencies have increased. This is to be expected because doing retries takes time.



## Timeouts

Linkerd can be used to limit how long to wait before failing outgoing requests to another service. These timeouts work by adding additional annotations to a service profile’s routes configuration.

1. Look at the current latency for requests from `webapp` to the `books`service:

   ```bash
   linkerd -n booksapp routes deploy/webapp --to svc/books
   ```

2. Requests to the `books` service’s `PUT /books/{id}.json` route include retries for when that service calls the `authors` service as part of serving those requests, as described in the previous section. This improves success rate, at the cost of additional latency. For the purposes of this demo, let’s set a 25ms timeout for calls to that route. To edit the `books` service profile, run:

   ```bash
   kubectl -n booksapp edit sp/books.booksapp.svc.cluster.local
   ```

   Update the `PUT /books/{id}.json` route to have a timeout:

   ```yaml
   spec:
     routes:
     - condition:
         method: PUT
         pathRegex: /books/[^/]*\.json
       name: PUT /books/{id}.json
       timeout: 25ms ### ADD THIS LINE ###
   ```

   Linkerd will now return errors to the `webapp` REST client when the timeout is reached. This timeout includes retried requests and is the maximum amount of time a REST client would wait for a response.

3. Run `routes` to see what has changed: 

   ```bash
   linkerd -n booksapp routes deploy/webapp --to svc/books -o wide
   ```

   We can see that the timeouts are working by observing that the effective success rate for our route has dropped below 100%.


# Lab 3: Canary Deployments

## Install and Configure Flagger

1. Ensure your `kubectl` is version 1.14 or greater or install Kustomize (https://github.com/kubernetes-sigs/kustomize)

2. Install Flagger in the `linkerd` namespace:

   ```
   kubectl apply -k github.com/weaveworks/flagger//kustomize/linkerd
   ```

3. Create a `test` namespace and enable Linkerd proxy injection:

   ```
   kubectl create ns test
   kubectl annotate namespace test linkerd.io/inject=enabled
   ```

4. Install the load testing service to generate traffic during the canary analysis:

   ```
   kubectl apply -k github.com/weaveworks/flagger//kustomize/tester
   ```

5. Create a deployment and a horizontal pod autoscaler:

   ```
   kubectl apply -k github.com/weaveworks/flagger//kustomize/podinfo
   ```

6. Create a canary custom resource for the `podinfo` deployment. Create `podinfo-canary.yaml` and add the following:

   ```
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

   ```
   kubectl apply -f ./podinfo-canary.yaml
   ```

   When the canary analysis starts, Flagger will call the pre-rollout webhooks before routing traffic to the canary. The canary analysis will run for five minutes while validating the HTTP metrics and rollout hooks every half a minute. Flagger implements a control loop that gradually shifts traffic to the canary while measuring key performance indicators like HTTP requests success rate, requests average duration and pod health. Based on analysis of the KPIs a canary is promoted or aborted. After a couple of seconds Flagger will create the canary objects. 

   After five minutes the `podinfo` deployment will be scaled to zero and the traffic to `podinfo.test` will be routed to the primary pods. During the canary analysis, the `podinfo-canary.test` address can be used to directly target the canary pods.



## Trigger a Canary Deployment

Canary deployments can be triggered by changes in any of the following objects:

- Deployment PodSpec (container image, command, ports, env, resources, etc)
- ConfigMaps mounted as volumes or mapped to environment variables
- Secrets mounted as volumes or mapped to environment variables

1. Trigger a canary deployment by updating the pod image: 

   ```
   kubectl -n test set image deployment/podinfo \
   podinfod=quay.io/stefanprodan/podinfo:1.7.1
   ```

2. Flagger detects that the deployment revision changed and starts a new rollout. View the rollout by describing the pod: 

   ```
   kubectl -n test describe canary/podinfo
   ```

3. Monitor all canaries with: 

   ```
   watch kubectl get canaries --all-namespaces
   ```



## Automated Rollback 

During the canary analysis you can generate HTTP 500 errors and high latency to test if Flagger pauses and rolls back the faulted version.

1. Trigger another canary deployment: 

   ```
   kubectl -n test set image deployment/podinfo \
   podinfod=quay.io/stefanprodan/podinfo:1.7.2
   ```

2. Exec into the `loadtester` pod: 

   ```
   kubectl -n test exec -it flagger-loadtester-xx-xx sh
   ```

3. Generate HTTP 500 errors: 

   ```
   watch -n 1 curl http://podinfo-canary.test:9898/status/500
   ```

4. In another shell generate latency: 

   ```
   watch -n 1 curl http://podinfo-canary.test:9898/delay/1
   ```

5. When the number of failed checks reaches the canary analysis threshold, the traffic is routed back to the primary, the canary is scaled to zero and the rollout is marked as failed. View the status of the rollout with: 

   ```
   kubectl -n test describe canary/podinfo
   ```



## Custom Metrics

The canary analysis can be extended with Prometheus queries. 

1. We can define a check for HTTP 404 not found errors. Edit the `canaryAnalysis` of the `podinfo` deployment: 

   ```
   kubectl edit deployment/podinfo 
   ```

    Add the following metric:

   ```
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

   ```
   kubectl -n test set image deployment/podinfo \
   podinfod=quay.io/stefanprodan/podinfo:1.7.3
   ```

3. In another shell generate 404's:

   ```
   watch -n 1 curl http://podinfo-canary:9898/status/404
   ```

4. Watch the Flagger logs to see the canary failure and rollback: 

   ```
   kubectl -n linkerd logs deployment/flagger -f | jq .msg
   ```


# Lab 4: Failure Injection 

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

   ```
   linkerd stat deploy
   ```

   We should now see the application is healthy and has a 100% success rate. 

4. Now we can create our error service. Here I will use NGINX configured to respond only with HTTP status code 500. Create a file called `error-injector.yaml`

   ```
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

   ```
   kubectl apply -f error-injector.yaml
   ```

6. Now we can create a traffic split resource which will direct 10% of the books service to the error service. Create a file called `error-split.yaml`:

   ```
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

   ```
   kubectl apply -f error-split.yaml
   ```

8. View the error rate:

   ```
   linkerd routes deploy/webapp --to service/books
   ```

9. Explore how the application handles these errors:

   ```bash
   kubectl port-forward deploy/webapp 7000 & 
   open http://localhost:7000
   ```

   If you refresh the page a few times, you will sometimes see an internal server error page.

10. Restore the health of the application by deleting the traffic split resource:

    ```
    kubectl delete trafficsplit/error-split
    ```

11. To remove the books app and the `booksapp` namespace from your cluster, run:

    ```
    curl -sL https://run.linkerd.io/booksapp.yml \
      | kubectl -n booksapp delete -f - \
      && kubectl delete ns booksapp
    ```



# Lab 5: Debugging Your Service

## Install the Sample Application 

To get a feel for how Linkerd would work for one of your services, you can install a demo application. The emojivoto application is a standalone Kubernetes application that uses a mix of gRPC and HTTP calls to allow the users to vote on their favorite emojis.

1. Install the emojivoto application into the `emojivoto` namespace by running:

   ```
   curl -sL https://run.linkerd.io/emojivoto.yml \
     | kubectl apply -f -
   ```

2. Before we mesh it, let’s take a look at the app. If you’re using [Docker Desktop](https://www.docker.com/products/docker-desktop) at this point you can visit [http://localhost](http://localhost/) directly. If you’re not using Docker Desktop, we’ll need to forward the `web-svc` service. To forward `web-svc` locally to port 8080, you can run:

   ```
   kubectl -n emojivoto port-forward svc/web-svc 8080:80
   ```

3. Now visit [http://localhost:8080](http://localhost:8080/). Voila! The emojivoto app in all its glory.

   Clicking around, you might notice that some parts of the emojivoto application are broken. If you click on a doughnut emoji, you will get a 404 page. These errors are intentional and we can use Linkerd to identify the problem. 

4. Add Linkerd to emojivoto by running:

   ```
   kubectl get -n emojivoto deploy -o yaml \
     | linkerd inject - \
     | kubectl apply -f -
   ```

   This command retrieves all of the deployments running in the `emojivoto` namespace, runs the manifest through `linkerd inject`, and then reapplies it to the cluster. The `linkerd inject`c ommand adds annotations to the pod spec instructing Linkerd to add the data plane’s proxy to the pod spec as a container.

5. Verify that everything worked the way it should with the data plane with:

   ```
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

   ```
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

[Flagger](<https://docs.flagger.app/>)




