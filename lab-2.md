# Lab 2: Demo Application

## Summary

This lab walks through the process of deploying a sample application and adding the Linkerd side car proxy. After Linkerd has been added to the application you will learn how to use it to debug an error in the application. Next, you will learn how to use service profiles to gain valuable information about traffic routing. Finally, you will use Linkerd to configure retries and timeouts for the application.

### What Do You Need?

* A Kubernetes cluster 
* A machine with the Linkerd CLI installed

## Install the Demo Application

1. Install the books app to your cluster with:

   ```text
   kubectl create ns booksapp && \
     curl -sL https://run.linkerd.io/booksapp.yml \
     | kubectl -n booksapp apply -f -
   ```

   This command creates a `booksapp` namespace for the demo application, downloads its Kubernetes resource manifest, and uses `kubectl` to apply it to your cluster.

2. Verify the installation with:

   ```text
   kubectl -n booksapp get all
   ```

   You should see deployments for `authors`, `books`, `traffic`, and `webapp`.

3. Access the application via portforward by running:

   ```text
   kubectl -n booksapp port-forward svc/webapp 7000 &
   ```

   Navigate to [http://localhost:7000/](http://localhost:7000/)

![bookinfo](.gitbook/assets/bookinfo.png)

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



   ```text
      linkerd dashboard &
   ```

2. The **Overview** tab will show all of the services running in the `booksapp` namespace. There will be success rate, requests per second, and latency percentiles. Notice that the success rate for `webapp` is not 100%. This is because the traffic generator is submitting new books. You can do the same thing yourself and push that success rate even lower. Click on `webapp` in the Linkerd dashboard for a live debugging session.

![overview](.gitbook/assets/overview.png)

3. On the **Detail** view for the `webapp` service you’ll see that `webapp` is taking traffic from `traffic` \(the load generator\), and it has two outgoing dependencies: `authors` and `book`. One is the service for pulling in author information and the other is the service for pulling in book information. We can see that while the `authors` service has a 100% success rate the `books` service is less than 100%. A failure in the dependent `books` service may be the cause of the errors that `webapp` is returning.

![detail](.gitbook/assets/detail.png)

4. Scroll down the page to see the live list of all traffic endpoints that `webapp` is receiving. The information on this page is populated based on the `linkerd top` command, which is used to display sorted information about live traffic.

![top](.gitbook/assets/top.png)

5. Inbound traffic coming from the `webapp` service going to the `books` service is failing a significant percentage of the time. This may explain why `webapp` is throwing intermittent failures. The information on this page is populated based on the `linkerd tap` command, which is used to listen to a traffic stream. Click on the 🔬 icon to look at the request and response stream.

![tap](.gitbook/assets/tap.png)

6. Many of these requests are returning HTTP 500 errors. Click on the **∨** in the left most column of any request to see additional details.

![tap\_detail](.gitbook/assets/tap_detail.png)

This shows how to diagnose an intermittent issue that affecting a single route. You now have everything you need to open a bug report explaining exactly what the root cause is. If the `books` service was your own, you know exactly where to look in the code.

## Service Profiles

[Service profiles](https://linkerd.io/2/features/service-profiles/) define the routes that you’re serving and, among other things, allow for the collection of metrics on a per route basis. The information is stored in Prometheus and can be queried at a later time.

1. One of the easiest ways to get service profiles setup is by using existing [OpenAPI \(Swagger\)](https://swagger.io/docs/specification/about/) specs. This demo has published specs for each of its services. You can create a service profile for `webapp` by running:

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

   Routes are simple conditions that contain a method \(`GET` for example\) and a regex to match the path. This allows you to group REST style resources together instead of seeing a huge list. The names for routes can be whatever you’d like. For this demo, the method is appended to the route regex.

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

   ```text
   linkerd -n booksapp tap deploy/webapp -o wide | grep req
   ```

   These metrics come from the [`linkerd route`](https://linkerd.io/2/reference/cli/routes/) command, which is used to display route stats. A similar command, [`linkerd stat`](https://linkerd.io/2/reference/cli/stat/) is used to display traffic stats about one or many resources. To see the metrics that have accumulated so far, run:

   ```text
   linkerd -n booksapp routes svc/webapp
   ```

   This will output a table of all the routes observed. The `[DEFAULT]` route is a catch all for anything that does not match the service profile.

5. Profiles can also be used to observe _outgoing_ requests as well as _incoming_ requests. To do that, run:

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

   ```text
   kubectl -n booksapp edit sp/authors.booksapp.svc.cluster.local
   ```

   And adding `isRetryable` to the troubled `HEAD /authors/{id}.json` route in the deployment spec:

   ```text
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

## Additional Reading

[Main Linkerd 2 GitHub Repo](https://github.com/linkerd/linkerd2)

[Linkerd Website](https://github.com/linkerd/website)

