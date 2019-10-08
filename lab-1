
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
   
## Additional Reading 

[Main Linkerd 2 GitHub Repo](https://github.com/linkerd/linkerd2)

[Linkerd Website](https://github.com/linkerd/website)

