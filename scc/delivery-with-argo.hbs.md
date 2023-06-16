# Use Gitops delivery with ArgoCD (alpha)

This topic explains how you can deliver Carvel `Packages`, created by the Carvel Package Supply Chains, from a GitOps repository to one or more run clusters using ArgoCD for Supply Chain Choreographer.

## <a id="prerecs"></a> Prerequisites

To use Gitops Delivery with ArgoCD, you must complete the following prerequisites:

- Create a `Workload` that uses either the `source-to-url-package` or
  `basic-image-to-url-package` Carvel Package Supply Chain. See the [Carvel
  documentation](./carvel-package-supply-chain.hbs.md). You must have at least
  one Carvel `Package` generated by this `Workload` stored in your GitOps
  repository.
- Have at least one run cluster. Run clusters serve as your deployment
  environments. They can either be Tanzu Application Platform clusters, or
  Kubernetes clusters, but they must have kapp-controller and Contour
  installed. See the [Carvel documentation](https://carvel.dev/kapp-controller/)
  and the [Contour documentation](https://projectcontour.io/).
- Create a build cluster that has network access to your run
  clusters to use a build cluster to control the deployment on all the run
  clusters. You must also install ArgoCD.
  If you intend to deploy directly on the run cluster without a build cluster,
  a build cluster is only necessary for building the package.

## <a id="run-cluster-ns"></a> Set up run cluster namespaces

Each run cluster must have a namespace and `ServiceAccount` with the correct permissions to deploy the Carvel `Packages`.

If your run cluster is a Tanzu Application Platform cluster, see [Set up developer namespaces to use installed packages](../install-online/set-up-namespaces.hbs.md).

If your run cluster is not a Tanzu Application Platform cluster, create a namespace and `ServiceAccount` with the following permissions:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: <run-cluster-ns>
  name: app-cr-role
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["configmaps", "services"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "create", "update", "delete"]
```

## <a id="create-carvel"></a> Create Carvel PackageInstalls and secrets

For each Carvel `Package` and run cluster, you must create a Carvel `PackageInstall` and a `Secret`. The Carvel `PackageInstall` and the `Secret` are stored in your GitOps repository and deployed to run clusters by FluxCD.

The following example shows a GitOps repository structure after completing this section:

```console
app.default.tap/
  packages/
    20230321004057.0.0.yaml  # Package
  staging/
    packageinstall.yaml      # PackageInstall
    params.yaml              # Secret
  prod/
    packageinstall.yaml      # PackageInstall
    params.yaml              # Secret
```

For each run cluster:

1. Create a `Secret` that has the values for each `Package` parameter. You can view the configurable properties of the `Package` by inspecting the `Package` CR’s valuesSchema, or in the [Carvel Package Supply Chains documentation](./carvel-package-supply-chain.hbs.md). Store the `Secret` in your GitOps repository at `<package_name>/<run_cluster>/params.yaml`.

   > **Note** You can skip this step to use the default parameter values.

   ```yaml
   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: app-values
   stringData:
     values.yaml: |
       ---
       replicas: 2
       hostname: app.mycompany.com
   ```

2. Create a `PackageInstall`. Reference the `Secret` you created earlier. Store the `PackageInstall` in your GitOps repository at `<package_name>/<run_cluster>/packageinstall.yaml`.

   > **Note** If you skipped creation of the `Secret`, omit the `values` key.

   ```yaml
   ---
   apiVersion: packaging.carvel.dev/v1alpha1
   kind: PackageInstall
   metadata:
     name: app
   spec:
     serviceAccountName: <run-cluster-ns-sa> # ServiceAccount on run cluster with permissions to deploy Package, see "Set up run Cluster Namespaces"
     packageRef:
       refName: app.default.tap # name of the Package
       versionSelection:
         constraints: 20230321004057.0.0 # version of the Package
     values:
     - secretRef:
         name: app-values # Secret created in previous step
   ```

   > **Note** To continuously deploy the latest version of your `Package`, you can set `versionSelection.constraints: >=0.0.0`

3. Push the `PackageInstalls` and `Secrets` to your GitOps repository.

## <a id="create-argo"></a> Create an ArgoCD application on the Build cluster

Configure ArgoCD on the Build cluster to deploy your `Packages`, `PackageInstalls`, and `Secrets` to each run cluster:

1. Register a cluster's credentials to Argo CD. Thi is only necessary when deploying to an external cluster.
  - First list all clusters contexts in your current kubeconfig:

   ```console
   kubectl config get-contexts -o name
   ```

  - Choose a context name from the list and supply it to the argocd cluster. This command installs a ServiceAccount, argocd-manager, into the kube-system namespace of that kubectl context, binding the service account to an admin-level ClusterRole. Argo CD uses this service account token to perform its management tasks, such as deployment and monitoring. 
  
  For example, for `run-cluster1` context, run:

   ```console
   argocd cluster add run-cluster-1
   ```

    > **Note** You can modify the rules of the argocd-manager-role role so that
    it only has create, update, patch, delete privileges to a limited set of
    namespaces, groups, kinds. However get, list, and watch privileges are required
    at the cluster-scope.

1. Create an application from a Git repository.

  - Set the current namespace to argocd:

   ```console
    kubectl config set-context --current --namespace=argocd
   ```

  - Create a hello-world-app:

   ```console
    argocd app create hello-world-app --repo https://github.com/mycompany/gitops-repo
   ```

1. Deploy the application.

  - After you create the application, you can view its status:

   ```console
    argocd app get hello-world-app
   ```

  The output is similar to the following:

    ```console
        Name:               hello-world-app
        Server:             https://kubernetes.default.svc
        Namespace:          default
        URL:                https://10.97.164.88/applications/hello-world-app
        Repo:               https://github.com/mycompany/gitops-repo.git
        Target:
        Path:               hello-world-app
        Sync Policy:        <none>
        Sync Status:        OutOfSync from  (1ff8a67)
        Health Status:      Missing

        GROUP  KIND        NAMESPACE  NAME                 STATUS     HEALTH
        apps   Deployment  default    hello-world-app-dep  OutOfSync  Missing
               Service     default    hello-world-app-svc  OutOfSync  Missing
        The application status is initially in OutOfSync state since the application has yet to be deployed, and no Kubernetes resources have been created. To sync (deploy) the application, run:

    ```
  This command retrieves the manifests from the repository and performs a kubectl apply. The hello-world-app app is running and you can now view its resource components, logs, events, and health status.

   ```console
    argocd app sync hello-world-app
   ```

## <a id="verify-install"></a> Verify installation

To verify your installation:

1. On your Build cluster, confirm that your FluxCD GitRepository and Kustomizations are reconciling:

   ```console
   kubectl get gitrepositories,kustomizations -A
   ```

2. Target a run cluster. Confirm that all Packages from the GitOps repository are deployed:

   ```console
   kubectl get packages -A
   ```

3. Target a run cluster. Confirm that all PackageInstalls are reconciled:

   ```console
   kubectl get packageinstalls -A
   ```

Now you can access your application on each run cluster.