## Setup

- Tabs to open
    - [https://cloudblogs.microsoft.com/opensource/2020/12/15/introducing-cluster-api-provider-azure-capz-kubernetes-cluster-management/](https://cloudblogs.microsoft.com/opensource/2020/12/15/introducing-cluster-api-provider-azure-capz-kubernetes-cluster-management/)
    - [https://kind.sigs.k8s.io/docs/user/quick-start/#installation](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
    - [https://cluster-api.sigs.k8s.io/user/quick-start.html](https://cluster-api.sigs.k8s.io/user/quick-start.html)
    - [https://github.com/kubernetes-sigs/cluster-api-provider-azure/tree/master/templates](https://github.com/kubernetes-sigs/cluster-api-provider-azure/tree/master/templates)
    - [https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/111a0de90b575fab1842d487492f01a316c944e9/templates/cluster-template.yaml](https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/111a0de90b575fab1842d487492f01a316c944e9/templates/cluster-template.yaml)
    - [https://cluster-api.sigs.k8s.io/user/personas.html](https://cluster-api.sigs.k8s.io/user/personas.html)
    - [https://capz.sigs.k8s.io/topics/getting-started.html#prerequisites](https://capz.sigs.k8s.io/topics/getting-started.html#prerequisites)
    - [https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/book/src/tasks/using-kustomize.md](https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/book/src/tasks/using-kustomize.md)
    - [https://forms.office.com/Pages/ResponsePage.aspx?id=v4j5cvGGr0GRqy180BHbR7CFOqLZhpRKk6-mBu7L1GhUM1o0SjZKTVAyOEFVWlNLMjFYR05WNEJaUC4u](https://forms.office.com/Pages/ResponsePage.aspx?id=v4j5cvGGr0GRqy180BHbR7CFOqLZhpRKk6-mBu7L1GhUM1o0SjZKTVAyOEFVWlNLMjFYR05WNEJaUC4u)

## ANNOUNCE

- Now that we've learned about Cluster API (CAPI) and Cluster API for Azure (CAPZ), let's get hands-on
- "K8S Clusters are like Lay's Potato Chips - Once you pop, you can't stop"
- In this walkthrough, you will learn how to:
    - Create a Kind cluster
    - Initialize CAPI Management Cluster
    - Create a CAPI Workload Cluster using CAPZ
- Show BLOG + Arc Jumpstart
    - [https://cloudblogs.microsoft.com/opensource/2020/12/15/introducing-cluster-api-provider-azure-capz-kubernetes-cluster-management/](https://cloudblogs.microsoft.com/opensource/2020/12/15/introducing-cluster-api-provider-azure-capz-kubernetes-cluster-management/)
    - [https://azurearcjumpstart.io/azure_arc_jumpstart/azure_arc_k8s/cluster_api/capi_azure/](https://azurearcjumpstart.io/azure_arc_jumpstart/azure_arc_k8s/cluster_api/capi_azure/)
    - This session will be a guided walkthrough of CAPI Quickstart and some use cases
        - [https://cluster-api.sigs.k8s.io/user/quick-start.html](https://cluster-api.sigs.k8s.io/user/quick-start.html)
- Re-emphasize prereqs
    - **Mac or Linux (WSL or Azure VM)**
    - Kind v0.10.0 [https://kind.sigs.k8s.io/docs/user/quick-start/#installation](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
    - Cluster CTL: [https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl)
- Will focus on self-managed k8s cluster
    - Replacing AKS-engine
- Future will allow creating Managed (AKS) clusters

## CONCEPTS

- Mgmt cluster - In this case, Kind.  This is control plane for fleet of clusters
    - How do you manage 100-1000's of clusters?  Like this!
    - Using KIND because it's really easy.
- Workload cluster - K8S Cluster in Azure
- kubeadm - community accepted tool for bootstrapping nodes that are going to be joined to create a self-managed K8S cluster

## CHECKPOINT

```jsx
➜   kind --version
kind version 0.10.0                                                                                                                                       [0.0s]
```

```jsx
➜   clusterctl version
clusterctl version: &version.Info{Major:"0", Minor:"3", GitVersion:"v0.3.15", GitCommit:"b900c6f89f3d433c32db1aa269f77f004a83cc4f", GitTreeState:"clean", BuildDate:"2021-03-30T16:14:03Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"darwin/amd64"}                                                                [0.0s]
```

## WALKTHROUGH

- Create kind cluster
    - `kind create cluster`
    - This is management cluster (bootstrap cluster) which will be used to provision the desired/workload cluster
    - 2-3m
    - Showcase kind cluster
        - `kubectl get nodes`
        - `docker ps`
        - `kubectl cluster-info`
    - Lifecycle:  Mgmt cluster will be up as long as the workload cluster
        - Example:  It will update on the CRD's
        - Acts as ongoing reconciler
        - Can have multi-mgmt clusters and move/pause clusters
    - Mgmt Cluster **Resources** are "pet", Kind cluster can be "cattle"
    - Mgmt Cluster CAN be pet.  But the Resources are "pet" and can be moved
    - Maybe ask upstream about Best Practices and lifecycle
        - e.g. AKS Secure Baseline Cluster can be codified into CAPI Resources
        - Interested in feedback from field for Best Pract. to influence this
- Init the mgmt cluster (1-2m)
    - NOTE:  Using Env vars.  Can store into a yaml
    - NOTE: Using "contributor" role.  Which is NOT least priv.  But this is good for now

```jsx
export AZURE_SUBSCRIPTION_ID=$(az account show -o json | jq -r '.id')
export AZURE_TENANT_ID=$(az account show -o json | jq -r '.tenantId')
# If using Managed identities, need to be Owner (for now)
az ad sp create-for-rbac --role contributor
export AZURE_CLIENT_ID=...
export AZURE_CLIENT_SECRET=...
export AZURE_SUBSCRIPTION_ID_B64="$(echo -n "$AZURE_SUBSCRIPTION_ID" | base64 | tr -d '\n')"
export AZURE_TENANT_ID_B64="$(echo -n "$AZURE_TENANT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_ID_B64="$(echo -n "$AZURE_CLIENT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_SECRET_B64="$(echo -n "$AZURE_CLIENT_SECRET" | base64 | tr -d '\n')"
clusterctl init --infrastructure azure
kubectl get pods,deployments -A
```

- Explore the mgmt cluster

```jsx
# wait for all pods to be Running
watch kubectl get pods,deployments -A
docker ps
kubectl get nodes -o wide
kubectl cluster-info
```

- Generate cluster config
    - NOTE: The cluster config describes our cluster.
        - API Model:AKS Engine::Cluster Template:Cluster API
            - Instead of JSON, using YAML
            - These are **living resources** that live in Management cluster instead of Day 1 snapshot of ARM templates
    - Each provider can provide different flavors for templates (diff. scenarios, etc., ipv6, GPU)
        - [https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/111a0de90b575fab1842d487492f01a316c944e9/templates/cluster-template.yaml](https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/111a0de90b575fab1842d487492f01a316c944e9/templates/cluster-template.yaml)
        - As user, you would probably want a combination of flavors
    - CAPI TEAM: Talk about different scenarios
        - Windows, Nvidia, ipv6, aad, etc.
        - CAPI team is working on expanding these
    - Now we will create the Cluster Config

    ```jsx
    clusterctl config cluster capi-quickstart \
      --kubernetes-version v1.18.16 \
      --control-plane-machine-count=3 \
      --worker-machine-count=3 \
      > capi-quickstart.yaml
    ```

- Explore cluster config
    - `vi capi-quickstart.yaml`
    - NOTE: Nothing has been created yet.
        - These are high level constructs for k8s cluster
    - Clusters def shows control plane, CIDR, nfra of type "AzureCluster"
    - AzureCluster has SubID
    - KubeadmControlPlane has fs, etcd data
    - AzureMachineTemplate which has info about VMSize env var
    - NOTE:  Feels a lot like ARM, but with K8S touch

## QUESTION

- This made a lot of assumptions about our design.
- Where did those assumptions come from?  Flavors
    - [https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/111a0de90b575fab1842d487492f01a316c944e9/templates/cluster-template.yaml](https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/111a0de90b575fab1842d487492f01a316c944e9/templates/cluster-template.yaml)
- Q: What if we want to change up the design?
    - A: Introduce concept of flavors
    - [https://github.com/kubernetes-sigs/cluster-api-provider-azure/tree/master/templates](https://github.com/kubernetes-sigs/cluster-api-provider-azure/tree/master/templates)
    - These are just examples
- Q: What if no single flavor fit my needs?
    - A: Can build your own and use kustomize + DIY flavors
    - Same concept as your Pod/Deployment template.  There's a template and you need to customize to your needs
    - Can use `kubectl explain AzureCluster` to pick and choose your own journey
        - `kubectl explain AzureCluster.spec`
- Apply the workload cluster (5s)
    - NOTE This applies the CRD, which will be quick (5s), but in true K8S fashion, you're specifying what you want and K8S is trying to reconcile by creating the actual cluster

    ```jsx
    kubectl apply -f capi-quickstart.yaml
    ```

- Check the cluster provisioning status (~10-12 min)

    Control plane is up in 3 min.

    As soon as control plane is up you can start applying

    Cluster Infra  = NOT Compute

    Machines = VM's

    ```jsx
    ➜   kubectl get cluster --all-namespaces
    + kubectl get cluster --all-namespaces
    NAMESPACE   NAME              PHASE
    default     capi-quickstart   Provisioned                                                                                                                           [0.1s]

    # NOTE:  Provisioned does NOT mean ready

    # This shows the API server
    ➜   kubectl get kubeadmcontrolplane --all-namespaces
    + kubectl get kubeadmcontrolplane --all-namespaces
    NAMESPACE   NAME                            INITIALIZED   API SERVER AVAILABLE   VERSION    REPLICAS   READY   UPDATED   UNAVAILABLE
    default     capi-quickstart-control-plane   true                                 v1.18.16   3                  3         3                                                [0.1s]

    ➜   clusterctl describe cluster capi-quickstart
    NAME                                                                READY  SEVERITY  REASON  SINCE  MESSAGE
    /capi-quickstart                                                    True                     3s
    ├─ClusterInfrastructure - AzureCluster/capi-quickstart              True                     6m56s
    ├─ControlPlane - KubeadmControlPlane/capi-quickstart-control-plane  True                     3s
    │ └─3 Machines...                                                   True                     4m1s   See capi-quickstart-control-plane-d6szj, capi-quickstart-control-plane-f85zz, ...
    └─Workers
      └─MachineDeployment/capi-quickstart-md-0
        └─3 Machines...                                                 True                     2m3s   See capi-quickstart-md-0-855c5bfccf-6hhtn, capi-quickstart-md-0-855c5bfccf-j6b52, ...                                                                                                                                                                 [0.1s]

    ```

    - NOTE: Control plane node is up, but unavailable!
    - Will get to that in a moment
- Look in Azure Portal
    - Go to capi-quickstart RG
    - See OK + ETCD disks, NIC, RT and 6VM (3 control plane, 3 workload) and 2 IP (API and egress)
    - Make note of "pip-capi-quickstart-apiserver" DNS Name
- Dig into other cluster resources
    - Can look at all of the resources from `capi-quickstart.yaml`

```jsx
kubectl describe cluster
kubectl describe AzureCluster
kubectl describe AzureMachineTemplate # shows desired state, not actual
kubectl describe AzureMachines # has status
kubectl describe KubeadmConfigTemplate
kubectl get cluster-api
```

- PAUSE:  Ask upstream team for any commentary

```jsx
watch clusterctl describe cluster capi-quickstart
```

- STATUS: Cluster running, but not Ready.  Let's fix that
    - Need to start talking to KubeAPI
    - `clusterctl get kubeconfig capi-quickstart > capi-quickstart.kubeconfig`
    - `vi capi-quickstart.kubeconfig`
    - `kubectl --kubeconfig=./capi-quickstart.kubeconfig get nodes`
    - NOTE:  Typical Kubeconf, Node hostname
        - Go back and show "pip-capi-quickstart-apiserver" PIP DNS Name.  Is the same
- To fix Cluster not being ready:  Needs a CNI
    - CAPI TEAM:  Everyone else supports Calico, but not us.  Please explain

    ```jsx
    kubectl --kubeconfig=./capi-quickstart.kubeconfig \
      apply -f https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-azure/master/templates/addons/calico.yaml

    # WAIT ABOUT 1-2 minutes

    kubectl --kubeconfig=./capi-quickstart.kubeconfig get nodes
    ```

- NOTE
    - At this point, we're ready to R&R!  Install Istio, or your favorite Cat/Dog voting demo app
    - [https://github.com/lastcoolnameleft/kubernetes-examples](https://github.com/lastcoolnameleft/kubernetes-examples)
    - kubectl --kubeconfig=./capi-quickstart.kubeconfig apply -f [https://raw.githubusercontent.com/lastcoolnameleft/kubernetes-examples/master/service/podinfo](https://raw.githubusercontent.com/lastcoolnameleft/kubernetes-examples/master/service/podinfo)

## Great!  Now what?

- Two common use cases to consider, scaling and upgrading the cluster
- Scaling

    ```jsx
    vi capi-quickstart.yaml
    # Search for replicas
    # Change MachineDeployment replicas to 2.
    kubectl apply -f capi-quickstart.yaml
    # Watch scheduling disabled
    # Go to portal.  Should see removed

    # Alternative!
    kubectl scale md capi-quickstart-md-0 --replicas 3
    ```

- Upgrading
    - Considerations
        - Upgrade the CAPI tooling vs Upgrade K8S versions vs Upgrading Nodes
        - Upgrade the CAPI tooling
            - Rarely done, not going to cover as that's not interesting
        - Upgrading Nodes
            - CAPI philosophy is to use Golden Images!
        - Upgrading K8S versions
            - Now you have k8s cluster, described by k8s resources.  Can use same commands.
            - BIG WIN:  Don't have to use ARM/Bicep

            ```jsx
            vi capi-quickstart.yaml
            # KubeadmControlPlane ONLY edit and replace version: v1.18.16 → v1.18.17
            kubectl apply -f capi-quickstart.yaml

            # NOTE: Takes a few minutes, should see 1 additional VM started
            KUBECONFIG=capi-quickstart.kubeconfig kubectl get nodes
            #NOTE:  It upgraded 1 CP node and then all my worker nodes

            ➜  KUBECONFIG=capi-quickstart.kubeconfig   kubectl get nodes
            + kubectl get nodes
            NAME                                  STATUS   ROLES    AGE     VERSION
            capi-quickstart-control-plane-97cg5   Ready    master   25h     v1.18.16
            capi-quickstart-control-plane-hgtz6   Ready    master   8m53s   v1.18.17
            capi-quickstart-control-plane-lpbbh   Ready    master   25h     v1.18.16
            capi-quickstart-md-0-q5qwt            Ready    <none>   5m23s   v1.18.17
            capi-quickstart-md-0-tvsmt            Ready    <none>   8m56s   v1.18.17
            capi-quickstart-md-0-v5l4h            Ready    <none>   3m4s    v1.18.17                                                                                                  
            ```

## CLEANUP

- Don't do this yet, but to cleanup, 2 considerations: workload cluster, mgmt cluster.  In that order

```jsx
kubectl delete cluster capi-quickstart
clusterctl delete -i azure
kind delete cluster
az ad sp delete --id $AZURE_CLIENT_ID
```

- Summary
    - In session, we've done the following:
        - Created Mgmt Cluster
        - Created workload cluster
        - Scaled and upgraded workload cluster
    - Next session, will focus on GitOps Flux v2 which can help tie all of this together
