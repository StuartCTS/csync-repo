# csync-repo
Demonstration repo for use wth Google [Config sync](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/overview?hl=en) and [Policy Controller](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller?hl=en). These are both individual components of [Anthos Config Management](https://cloud.google.com/anthos-config-management/docs/overview?hl=en).

Conceptually, ConfigSync enables a Gitops style flow for managing configuration of GKE clusters. Configurations are stored as yaml in a a git repo, and an agent on the cluster continual syncs with the repo, detecting and applying any changes. Similarly, if discrepancies are noticed between the state of the cluster and the repo, it will automatically attempt to resolve these.

## Prerequisites
k8s cluster(s) with `kubectl` access

`gcloud/gsutil` access to a GCP project, with the ability to create/update roles on the clusters

Access to create/fork a github repo

## Installation

### git repo  
Fork this repo to your own github so you can make changes and see them reflected in the synced clusters.

### nomos 
`nomos` is a command line utility executable from Google to obtain the sync status of your clusters, as well as vet any config repo changes. This leverages your current `kubeconfig` to check the sync status of clusters using Config Sync

### Cluster setup and install
Create one or more k8s clusters which are accessible via kubectl. Note installing both ConfigSync and Policy Controller can use some significant cluster resources - hence you will need something larger than the micro instances; also, GKE Autopilot clusters are not suitable due to their constrained nature.

A typical GKE cluster config that works ok for this:
```
gcloud beta container --project "stuart-dev-example-01" clusters create "belgium" --zone "europe-west1-b" --no-enable-basic-auth --cluster-version "1.18.16-gke.502" --release-channel "regular" --machine-type "n1-standard-2" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "3" --no-enable-stackdriver-kubernetes --enable-ip-alias --network "projects/stuart-dev-example-01/global/networks/default" --subnetwork "projects/stuart-dev-example-01/regions/europe-west1/subnetworks/default" --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --enable-shielded-nodes --node-locations "europe-west1-b"
```


### Install the Config Management operator

Get the latest version of _either_ the config management or config sync operator: 
```
gsutil cp gs://config-management-release/released/latest/config-management-operator.yaml ~/tmp/config-management-operator.yaml
```
_or_
```
gsutil cp gs://config-management-release/released/latest/config-sync-operator.yaml config-sync-operator.yaml
```

Then apply the CRD appropriately, i.e.
```
kubectl apply -f config-sync-operator.yaml
```
Note technically ConfigManager is only valid for licensed Anthos subscriptions and Anthos GKE clusters - however it seems to work ok on standard GKE (and even local) clusters for demo purposes. Note if you choose COnfig Sync, you won't be able to install the Policy Controller, but all the rest will work the same. 

At this point you should namespaces and associated objects created in the cluster, but there will be no active syncing of resources - to action this, you need to setup access to the repo and configure the installation.

### Config repo setup
Make sure you have forked this repo to a suitable location. Note for simplicity tis demo assumes the use of a public repo - this just means we don't have to configure authentication for Config SYnc - however this would be most likely required in any realistic deployment scenario. If you need to use a private repo see here for configuration steps, see details [here](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/how-to/installing?hl=en#git-creds-secret) and add the appropriate info to the config management configuration.

### Configuring Sync
Config CSync is configured by applying a yaml based configuration - a minimal example is shown below:
```
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  clusterName: YOUR_CLUSTER_NAME
  git:
    syncRepo: https://github.com/YOUR_GITHUB/YOUR_REPO
    syncBranch: main 
    secretType: none
    policyDir: karate-corp
  policyController:
    enabled: true
```

Note the repo details and the fact a unique cluster name must be specified. Apply this to the cluster:
```
kubectl apply -f config-management.yaml
```

Make sure you are using `kubectl with the appropriate context - i.e. if using multiple clusters, use 

```
kubectl apply -f config-management.yaml --context=CLUSTER_NAME
```

*Note* only add the last two lines if you have install _Config Management_ (as opposed to _Config Sync_) and intend to use Policy Controller.

### Checking the install

At this point the Config Sync should be synchronizing objects stored in your configured repo - you can check ths via using 
``` 
nomos status
```

Note when first syncing, you may see various errors being reported until all elements have been synced. If after a while you are seeing the same error consistently, review the log and events - this is usually to do with:

- misconfiguration of the repo location/branch/folder/creds
- lack of quota/resource on the cluster

### Cluster registration  
For each of your enrolled clusters you should add a cluster registration to the `/clusterregistry folder of your repo (see e.g. [](karate-corp/clusterregistry/cluster-local.yaml) ). This will allow you to define cluster selectors based upon teh labels attached to these cluster definitions. Note the names _must_ correspond with those used when configuring config-sync on the cluster. 

In this repo, it is assumes there will be four clusters, three on GKE (`belgium`, `iowa` and `taiwan`) and one local cluster (`docker-desktop`). you will need something similar to demonstrate concepts such as locality constraints or configuration targeting via cluster selectors.

### Cluster selectors
There are two sample cluster selector types based upon the above sample clusters:

- location selectors, one for each cluster e.g.[](karate-corp/clusterregistry/clusterselector-location-iowa.yaml)
- an environment based selector, based upon an environment label set for each cluster e.g. [](karate-corp/clusterregistry/clusterselector-env-prod.yaml)

THese can be used with the correct annotations to apply configurations selectively to individual or groups of clusters

## Config Sync features

There are samples illustrating a number of cluster configuration and management features within this repo. Specific instructions listed below

### Namespace management
[hello](karate-corp/namespaces/hello/hello.yaml) defines a namespace - adding these to the repo will ensure the ns gets created on asuitable clusters. If you delete this, the ns will be re-created.

### Daemonset deployment
Config Sync can be used to apply any yaml as you would via `kubectl apply -f ...`. A [fluentd ElasticSearch daemonset deployment](karate-corp/namespaces/fluentd/fluentd-es.yaml) is included to illustrate this, deployed to a `fluentd` namespace. Note as configured this will be deployed on all clusters. 

If using Policy Controller to apply policies, make sure you add the `fluentd` ns to the list of exclusions.

### Locality enforcement
Objects defined in the `/cluster` folder have cluster scope and are generally applied to all clusters. However supposing we only want,sa, an auditor privilege to be available in a specific location? The [clusterrole-auditor](karate-corp/cluster/clusterrole-auditor.yaml) clusterrole is annotated to use the [cluster-belgium](karate-corp/clusterregistry/cluster-belgium.yaml) cluster selector -i.e. it will only be applied to clusters that satisfy that selector - in this case, the `belgium` cluster only.

### Abstract namespaces
Namespaces can be defined within an abstract namespace - as such they can inherit common configurations defined at that level. In the example, both blue and green team namespaces will automatically inherit the `ops-role` and associated binding.

### Namespace selectors
As above, an SRE admin role is defined in teh abstract namespaces. However, this includes a reference (via annotation) to the [sre-supported-selector](karate-corp/namespaces/abstract-karate-app-ns/sre-supported-selector.yaml), which selects only namespaces with the environment label matching 'prod. As such, the sre-admin role will only be present in the `greenteam` namespace, as this is labeled being in the prod env.

## PolicyController
Policy COntroller is an integrated version of OPA Gatekeeper, offered as part of ACM. It allows policies governing the behaviour of objects in the cluster to be authored and applied as YAML. Combined with ConfigSync, these can also be managed and deployed across clusters in a GitOps manner.

Policy Controller ships witha library of policy templates that are installed as default in the cluster as CRDs, i.e.

```
stuart@Stuarts-MacBook-Pro config-sync % kubectl get constrainttemplates
NAME                                      AGE
allowedserviceportname                    7d
destinationruletlsenabled                 7d
disallowedauthzprefix                     7d
gcpstoragelocationconstraintv1            7d
k8sallowedrepos                           7d
k8sblocknodeport                          7d
k8sblockprocessnamespacesharing           7d
k8scontainerlimits                        7d
k8scontainerratios                        7d
...
```

The templates are parametrized. Policies are applied by creating _constraints_ that are based upon these tempaltes, supplying parameters, e.g. a policy that requires namespaces to have a specific label:

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-owner
spec:
  enforcementAction: dryrun
  match:
    excludedNamespaces: ["kube-system", "gatekeeper-system"]
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels:
      - key: "owner"
```

The constraint references the `K8sRequiredLabels` template, specifying that objects of type 'namespace' must have the label 'owner' present in their definition, otherwise the constraint will be violated. Depending on the _enforcement_ of the policy, this will either be audited or blocked via the Policy Controller admission controller - ie. namespace creation will be prevented until the issue is remediated.

### Installation
As above via the COnfig Management configuration.

### Creating a policy
Simple namespace label policy as above - see [](samples/constraints/ns-must-have-owner-constraint.yaml).

```
kubectl apply -f samples/constraints/ns-must-have-owner-constraint.yaml
```

### Viewing policies

View all templates installed:
```
kubectl get constrainttemplates
```

All polices (constraints)
```
kubectl get constraints
```

### View detailed info on policy
```
kubectl describe k8srequiredlabels ns-must-have-owner
```

To audit all policies:

```
kubectl logs -n gatekeeper-system -l gatekeeper.sh/system=yes
```

## Apply policy via ConfigSync
Policies are cluster scoped objects - as such they should be copied to the `/clusters` folder
