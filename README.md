# Install Red Hat Advanced Cluster Management using the CLI

## Prerequisites
Requirements:
- OpenShift Container Platform version 4.12 or later is deployed (if installing RHACM 2.9) and you are logged in with the OpenShift CLI
- Your OpenShift cluster has access to the Red Hat Advanced Cluster Management operator in the OperatorHub catalog
- Your OpenShift permissions allow you to create namespaces
- Your allowedRegistries list includes registry.redhat.io/multicluster-engine and registry.redhat.io/rhacm2
- You have cloned this repository or downloaded the necessary files


## Install RHACM 2.9  

The latest version of RHACM that is compatible with OpenShift 4.12 is RHACM v2.9. For later versions of OpenShift, see the [support matrices](https://access.redhat.com/support/policy/updates/advanced-cluster-management) (bottom of the page) to determine which version of RHACM to install.

### Create the OperatorGroup  

Create acm-operator-group.yaml to configure the OperatorGroup resource. Each namespace can only have one operator group.  

```
oc apply -f acm-install/acm-operator-group.yaml
```

### Create the Operator Subscription  

Create acm-operator-subscription.yaml to configure an OpenShift subscription to the advanced-cluster-management operator.  
Replace the *channel* with the ACM release you will be installing.  

```
oc apply -f acm-install/acm-operator-subscription.yaml
```

### Create the MultiClusterHub custom resource  

Create multiclusterhub.yaml to configure the MultiClusterHub custom resource.  

```
oc apply -f acm-install/multiclusterhub.yaml
```

Wait for multiclusterhub to install. If this step fails with an error, it is possible that the necessary resources are still being created and applied. Wait a few minutes and try again.  

Run the following command to get the custom resource. Wait for the status to display *Running*.  

```
oc get mch -o=jsonpath='{.items[0].status.phase}'
```  

Once the MultiClusterHub is installed properly, the Advanced Cluster Management web console should be accessible for cluster administrators.  


## Import a managed cluster  

### Create a new project  

Create a new project with the name of the cluster you are importing.  

```
oc new-project <cluster-name>
```  

Perform all further actions from this namespace.  

### Create the file to define the managed cluster  

Replace <cluster-name> with the name of your cluster. When *cloud* and *vendor* are set to auto-detect, Advanced Cluster Management detects the cloud and vendor types automatically from the cluster that you are importing. You can optionally replace the values for *auto-detect* with the values for your cluster.  
**IMPORTANT**: The label *cluster.open-cluster-management.io/clusterSet* defines which ManagedClusterSet the cluster will be added to. This example provides one cluster labeled "dev" and one cluster labeled "prod."  
In this example, mc1 is assumed to be a *dev* cluster while mc2 is assumed to be a *prod* cluster.

```
oc apply -f managed-cluster/mc1/managed-cluster.yaml
```  

### Import the managed cluster by using the auto import secret  

To import a managed cluster, we will use the auto import secret method. It is also possible to import the cluster manually.  
The auto import secret method can be performed using either a reference to the *kubeconfig* file, or the kube API server and token pair of the cluster. For this example, we use the API and token pair.  

Replace <cluster-name> with your cluster name, and replace <Token> and <cluster-api-url> with your API token and URL respectively. Ensure this is created in the <cluster-name> namespace.  

```
oc apply -f managed-cluster/mc1/auto-import-secret.yaml
```

The secret should auto import, and the managed cluster should begin importing. This resource is automatically deleted once consumed.  
Wait for the managed cluster to display the *JOINED* and *AVAILABLE* status. This may take several minutes.  
```
oc get managedcluster <cluster-name>
```  
```
oc get pods -n open-cluster-management-agent
```  

### Enabling klusterlet add-ons for the managed clusters

For each managed cluster you wish to enable klusterlet add-ons for, apply the YAML file ensuring the correct cluster name and namespace is set. For any add-ons you wish to disable, set *enabled: false*.  

```
oc apply -f managed-cluster/mc1/klusterlet-addon-config.yaml
```

## Creating ManagedClusterSets

Clusters in RHACM are grouped by ClusterSets. These ClusterSets can be used to apply policies across groups of clusters. The "global" clusterset includes all managed clusters and cannot be deleted, edited, or updated. We will create two additional clustersets as an example, named "dev" and "prod." ManagedClusterSets automatically include any managed clusters found with the correct label. Since the managed cluster YAML definition includes the correct labels, the managed clusters will automatically be added to the new ManagedClusterSet once created with the proper name.  

In order to create a new ManagedClusterSet, apply the YAML file which creates the ManagedClusterSet and ManagedClusterSetBinding. Ensure they are named properly in order for the clusters to properly join the clustersets.
```
oc apply -f managed-cluster/clustersets/dev-managedclusterset.yaml
oc apply -f managed-cluster/clustersets/prod-managedclusterset.yaml
```  

If you are adding a managed cluster to an existing ManagedClusterSet, you may need the proper RBAC *clusterrole* entry to assign the permission for joining clustersets. See the YAML file for the definition.  
```
oc apply -f managed-cluster/clustersets/managedclusterset-clusterrole.yaml
```  

You can verify the clusters were added to the correct ManagedClusterSets by viewing the RHACM web console Infrastructure > ClusterSets and clicking on the cluster set name. Any managed clusters assigned to that cluster set should appear under "Cluster list."  


## Health Metrics

You can use Prometheus to expose metrics that are not exposed from the product console. The following procedures can be followed to expose metrics for both the hub and managed clusters.  

### Scraping the hub cluster  

The three files prefixed by "hub" will be applied to create a ServiceMonitor, Role, and associated RoleBinding for collecting services and exposing metrics on the hub cluster.  
```
oc apply -f health-metrics/hub-service-monitor.yaml  
oc apply -f health-metrics/hub-monitoring-role.yaml  
oc apply -f health-metrics/hub-monitoring-rolebinding.yaml
```  

### Scraping the managed cluster  

The three files prefixed by "mc" will be applied to create a ServiceMonitor, Role, and associated RoleBinding for collecting services and exposing metrics on the managed cluster.  
```
oc apply -f health-metrics/mc-service-monitor.yaml  
oc apply -f health-metrics/mc-monitoring-role.yaml  
oc apply -f health-metrics/mc-monitoring-rolebinding.yaml
```  

### Scraping the standalone cluster  

The three files prefixed by "sa" will be applied to create a ServiceMonitor, Role, and associated RoleBinding for collecting services and exposing metrics on the standalone cluster.  
```
oc apply -f health-metrics/sa-service-monitor.yaml  
oc apply -f health-metrics/sa-monitoring-role.yaml  
oc apply -f health-metrics/sa-monitoring-rolebinding.yaml
```  

## Governance 

### Creating policies

This section provides several example policies to enforce or inform according to a specific Placement, which will be created in the following section.  

**IMPORTANT**: All of these policies are to be created in the rhacm-policies namespace.
```
oc new-project rhacm-policies
```  

Policies:  
| Policy name | Example placement | Mode | Description |
| policy-namespace | global | Enforce | Ensures the "paasops" namespaces exists and creates it if it doesn't. |
| policy-etcdencryption | global | Enforce | Ensures etcd encryption is enabled on the cluster and enables it if it isn't. |
| policy-comp-operator | global | Enforce | Ensures the compliance operator is installed on the cluster and installs it if it isn't. |
| policy-clusterroles | dev | Inform | Verifies the cluster-level roles exist as specified and provides a warning if they don't. |
| policy-clusterrolebindings | dev | Inform | Verifies the clusterrolebindings exist as specified and provides a warning if they don't. |
| policy-fileintegrity-operator | prod | Enforce | Ensures the file integrity operator is installed on the cluster and installs it if it isn't. |  

**NOTE**: In RHACM 2.10+, the *OperatorPolicy* kind was introduced to allow operator policies such as policy-comp-operator and policy-fileintegrity-operator to be defined all within one module, rather than needing to define the OperatorGroup and Subscription independently.  

Run *oc apply* to create these policies. The Placements defined in the next section will define which ManagedClusterSets apply which policies.  
```
oc apply -f <policy>.yaml
```  

The policies set to *INFORM* will only provide a warning about policy violations. Policies set to *ENFORCE* will attempt to remediate any policy violations according to the provided configuration.  

### Placing policies using Placements  

Placements and PlacementBindings are used to select which clusters a specific PolicySet is applied to. We will create three placements following the table defined above.  

The *Placement* defines which ManagedClusterSets are targeted under the *clusterSets* spec.  
The *PlacementBinding* binds the Placement to the correct namespace, in our case, *rhacm-policies*.  
The *PolicySet* allows you to group multiple policies into one set. These will be used to apply policies individually to their designated lifecycle (dev, prod, or global).  

```
oc apply -f dev-placement.yaml
oc apply -f prod-placement.yaml
oc apply -f global-placement.yaml
```

After applying these placements, view the Governance page of the RHACM web console. The Overview tab will give you a general overview of which PolicySets and clusters are reporting violations.  
More granular information about the specific policies can be viewed from the Policies tab. Any violations can be clicked on to view the exact error details on why the policy is in violation.  

*NOTE*: If any clusters show "Unknown status" for policies, you may have to enable klusterlet addons for that cluster. See [Enabling klusterlet add-ons for the managed clusters](#enabling-klusterlet-add-ons-for-the-managed-clusters).  


