# Install Red Hat Advanced Cluster Management using the CLI

## Prerequisites
Requirements:
- OpenShift Container Platform version 4.12 or later is deployed (if installing RHACM 2.9) and you are logged in with the OpenShift CLI
- Your OpenShift cluster has access to the Red Hat Advanced Cluster Management operator in the OperatorHub catalog
- Your OpenShift permissions allow you to create namespaces
- Your allowedRegistries list includes registry.redhat.io/multicluster-engine
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

```
oc apply -f managed-cluster/managed-cluster.yaml
```  

### Import the managed cluster by using the auto import secret  

To import a managed cluster, we will use the auto import secret method. It is also possible to import the cluster manually.  
The auto import secret method can be performed using either a reference to the *kubeconfig* file, or the kube API server and token pair of the cluster. For this example, we use the API and token pair.  

Replace <cluster-name> with your cluster name, and replace <Token> and <cluster-api-url> with your API token and URL respectively. Ensure this is created in the <cluster-name> namespace.  

```
oc apply -f managed-cluster/auto-import-secret.yaml
```

The secret should auto import, and the managed cluster should begin importing. This resource is automatically deleted once consumed.  
Wait for the managed cluster to display the *JOINED* and *AVAILABLE* status. This may take several minutes.  
```
oc get managedcluster <cluster-name>
```  
```
oc get pods -n open-cluster-management-agent
```  

### Enable klusterlet add-ons for the managed clusters

For each managed cluster you wish to enable klusterlet add-ons for, apply the YAML file ensuring the correct cluster name and namespace is set. For any add-ons you wish to disable, set *enabled: false*.  

```
oc apply -f managed-cluster/klusterlet-addon-config.yaml
```

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

This section provides several example policies to enforce or inform according to a specific Placement.
