# Lab 1: Install Linkerd

## Summary

This lab walks through the process of installing the Linkerd CLI onto a host machine and installing the Linkerd control plane onto a Kubernetes cluster.

### What Do You Need?

* A Kubernetes cluster 
* A machine onto which you can install the Linkerd CLI

1. Download the Linkerd CLI:

   ```text
   curl -sL https://run.linkerd.io/install | sh
   ```

2. Add Linkerd to your path:

   ```text
   export PATH=$PATH:$HOME/.linkerd2/bin
   ```

3. Verify the CLI was properly installed with:

   ```text
   linkerd version
   ```

4. Validate your cluster is configured correct for Linkerd with:

   ```text
   linkerd check --pre
   ```

5. Generate a manifest file and apply it to your cluster with:

   ```text
   linkerd install | kubectl apply -f -
   ```

   In addition to creating the `linkerd` namespace, this command installs the following resources onto your Kubernetes cluster:

   * ClusterRole
   * ClusterRoleBinding
   * CustomResourceDefinition
   * MutatingWebhookConfiguration
   * PodSecurityPolicy
   * Role
   * RoleBinding
   * Secret
   * ServiceAccount
   * ValidatingWebhookConfiguration

6. Verify your install with:

   ```text
   linkerd check
   ```

7. See what components were installed:

   ```text
   kubectl -n linkerd get deploy
   ```

8. Navigate to the dashboard with:

   ```text
   linkerd dashboard
   ```

   You will see a dashboard containing all of the applications running in the data plane.

![overview](.gitbook/assets/overview.png)

## Additional Reading

[Main Linkerd 2 GitHub Repo](https://github.com/linkerd/linkerd2)

[Linkerd Website](https://github.com/linkerd/website)

