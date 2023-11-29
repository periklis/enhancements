---
title: multi-cluster-observability-addon
authors:
  - "@periklis"
  - "@JoaoBraveCoding"
  - "@pavolloffay"
  - "@iblancasa"
reviewers:
  - "@alanconway"
  - "@jcantrill"
  - "@xperimental"
  - "@jparker-rh"
  - "@berenss"
  - "@bjoydeep"
approvers:
  - "@alanconway"
  - "@jcantrill"
  - "@xperimental"
api-approvers:
  - "@alanconway"
  - "@jcantrill"
  - "@xperimental"
creation-date: 2023-11-28
last-updated: 2023-11-28
tracking-link:
  - https://issues.redhat.com/browse/OBSDA-356
  - https://issues.redhat.com/browse/OBSDA-393
  - https://issues.redhat.com/browse/LOG-4539
see-also:
  - None
replaces:
  - None
superseded-by:
  - None
---

# Multi-Cluster Observability Addon

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

Multi-Cluster Observability has been an integrated concept in Red Hat Advanced Cluster Management (RHACM) since its inception but only incorporates one of the core signals, namely metrics, to manage fleets of OpenShift Container Platform (OCP) based clusters (See [RHACM Multi-Cluster-Observability-Operator (MCO)](rhacm-multi-cluster-observability)). The underlying architecture of RHACM observability consists of a set of observability components to collect a dedicated set of OCP metrics, visualizing them and alerting on fleet-relevant events. It is an optional but closed circuit system applied to RHACM managed fleets without any points of extensibility.

The following enhancement seeks to bring a unified approach to collect and forward logs and traces from a fleet of OCP clusters based on the RHACM addon facility (based upon the Open Cluster Management (OCM) [addon framework](ocm-addon-framework)) by enabeling these signals events to land on centralized storage locations chosen and managed by a third-party. The multi-cluster observability addon is an optional addon for  RHACM and a day two companion for MCO but doesn't necessarily share any resources/configuration with it. It provides a unified installation approach of required dependencies (e.g. operators) and resources (custom resources, certificates, CA Bundles, configuration) on the fleet clusters to collect and forward logs and/or traces off them. The addon is called Multi Cluster Observability Addon (MCOA).

## Motivation

The main driver for the following work is to simplify and unify the installation of log and/or trace collection and forwarding on an RHACM managed fleet of OCP clusters. The core utility function of the addon is to install required operators (i.e. [Red Hat OpenShift Logging](ocp-cluster-logging-operator), [Red Hat OpenShift distributed tracing data collection](opentelemetry-operator)), pre-configured set of custom
resources (i.e. `Clusterlogging`, `ClusterLogForwarder`, `OpenTelemetryCollector`) and referenced resources (e.g. Secrets including authentication credentials, Certificate Authority Bundles). Thus, the log/trace collection and forwarding is controled centrally from a RHACM Hub cluster.

### User Stories

* As a fleet administrator I want to install a homogeneous log collection and forwarding on any set of managed OCP clusters.
* As a fleet administrator I want to install a homogeneous trace collection and forwarding on any set of managed OCP clusters.
* As a fleet administrator I want to centrally control authentication credentials for the target log storage endpoints.
* As a fleet administrator I want to centrally control authentication credentials for the target trace storage endpoints.

### Goals

* Provide an optional RHACM addon to control log and/or trace collection and forwarding on managed OCP clusters.
* Enable control of authentication and authorization of storage endpoints from the hub cluster.
* Optionally leverage hub-local/controlled TLS Certificate management.

### Non-Goals

* Provide out-of-the-box centralized management for log and/or trace storage.
* Provide central ingestion/querying of logs and/or traces via the MCO gateway.

## Proposal

The following sections describe in detail the required custom resources as well as the workflow to enable log and/or trace collection and forwarding on an RHACM managed fleet of OCP clusters.

### Workflow Description

The workflow implemented in this proposal enables fleet-wide log/trace collection and forwarding as follows:

1. The hub administrator registers the MCOA on RHACM using a dedicated `ClusterManagementAddOn` resource along with the addon `Deployment` and accompanying RBAC.
2. The hub administrator creates a default `ClusterLogForwarder` stanza for on the hub cluster's `openshift-logging` namespace to describe the list of target log storage outputs. This stanza will then be used as a template by MCOA when generating the `ManifestWorks` for each managed cluster.
3. The hub administrator creates a default `OpenTelemetryCollector` stanza for on the hub cluster's `openshift-opentelemetry-operator` namespace the list of target trace storage outputs. This stanza will then be used as a template by MCOA when generating the `ManifestWorks` for each managed cluster.
4. The hub administrator creates an `AddOnDeploymentConfig` resource on each managed cluster namespace on the hub cluster to describe general addon configuration parameters (e.g. Operator Subscription Channel names, TLS Certificates, Authentication endpoints, etc.)
5. The MCOA will render a `ManifestWorks` resource per cluster that consists of a rendered manifest list (i.e. OLM subscriptions, `ClusterLogForwarder`, `OpenTelemetryCollector`, accompanying `Secret`'s and `ConfigMap`'s).
6. The WorkAgentController on each managed cluster will apply each individual manifest from the `ManifestWorks` locally.

Joao notes/comments:
- Regarding the stanzas, assuming we have the CRDs installed on the cluster, will we not have to patch them to allow CRs to be created without certain fields?
- (Different issue but worth considering) Helm doesn't like to manage CRDs if we by chance add a CRD for this project we should consider how we will tell users to install & update it https://helm.sh/docs/chart_best_practices/custom_resource_definitions/
- Random idea, WDYT of having the stanzas be part of the configuration of the AddOn. This would solve the potentail problem of the CRDs but it also would avoid us running into the problem of the user having a newer version of the CRDs on the hub but selecting an older version of the operator to be installed on the managed clusters
- Regarding the templating I checked and its not clear if `AddOnDeploymentConfig` supports nested keys https://pkg.go.dev/open-cluster-management.io/addon-framework@v0.8.0/pkg/addonfactory#ToImageOverrideValuesFunc (Looks like it supports for image but we would also need support for other values). Otherwise we would have to build a helm values file that would allow each and every field that needs to be configured to live at top level...
- Regarding the secrets/configmaps part I'm a bit unsure on the many different config knobs that we might need to support. For instance IIUC you propose the use of an annotation to know what secret goes into which part of CLF but can we make something similar for OTC?


The following subscections will describe in detail the form and content of each individual resource mentioned in the above workflow. For the sake of simplicity we assume a single managed cluster named `managed-ocp-cluster-1` and the following examplary version of the `ClusterManagementAddon` resource available on the hub cluster:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
 name: multi-cluster-observability-addon
spec:
 addOnMeta:
   displayName: Multi Cluster Observability Addon
   description: "multi-cluster-observability-addon is the addon to configure spoke clusters to collect and forward logs/traces to a given set of outputs"
 supportedConfigs:
   # Describes the general addon configuration applicable for all managed clusters. It includes:
   # - Default subscription channel name for install the `Red Hat OpenShift Logging` operator on each managed cluster.
   # - Default subscription channel name for install the `Red Hat OpenShift distributed tracing data collection` operator on each managed cluster.
   - group: addon.open-cluster-management.io
     resource: addondeploymentconfigs
     defaultConfig:
       name: multi-cluster-observability-addon
       namespace: open-cluster-management

   # Describe per managed cluster sensitive data per target forwarding location, currently supported:
   # - TLS client certificates for mTLS communication with a log/trace storage location.
   # - Client credentials for password based authentication with a log/trace storage location.
   - resource: secrets

   # Describe per managed cluster auxilliary config per target forwarding location.
   - resource: configmaps

   # Describe the default log forwarding locations for each log type applied to all managed clusters.
   - group: logging.openshift.io/v1
     resource: ClusterLogForwarder
     defaultConfig:
       name: instance
       namespace: open-cluster-management

   # Describe the default trace forwarding locations for each OTEL receiver applied to all managed clusters.
   - group: opentelemetry.io/v1alpha1
     resource: OpentelemetryCollector
     defaultConfig:
       name: instance
       namespace: open-cluster-management
```

and a default `AddonDeploymentConfig` describing the operator subscriptions to be used on each managed cluster:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnDeploymentConfig
metadata:
  name: multi-cluster-observability-addon
  namespace: open-cluster-management
spec:
  customizedVariables:
    - name: cluster-logging-channel
      value: stable-5.8
    - name: otel-channel
      value: stable
```

#### Multi Cluster Log Collection and Forwarding

For all managed clusters the hub administrator is required to provide a single `ClusterLogForwarder` resource stanza that describes the forwarding configuration to a set of target log storage endpoints in the default namespace `open-cluster-management`. The following example resource describes a configuration for forwarding logs from cluster `managed-ocp-cluster-1` based on log type:

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: open-cluster-management
spec:
  outputs:
  - loki:
       labelKeys:
       - log_type
       - kubernetes.namespace_name
       - kubernetes.pod_name
       - openshift.cluster_id
    name: application-logs
    type: loki
  - name: cluster-logs
    type: cloudwatch
    cloudwatch:
      groupBy: logType
      region: us-east-1
  pipelines:
  - inputRefs:
    - application
    name: application-logs
    outputRefs:
    - application-logs
  - inputRefs:
    - audit
    - infrastructure
    name: cluster-logs
    outputRefs:
    - cluster-logs
```

For each managed cluster the addon configuration looks as follows referencing per cluster. It references managed specific configuration resources, i.e. Secrets/ConfigMaps per `ClusterLogForwarder` output:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: multi-cluster-observability-addon
  namespace: managed-ocp-cluster-1
spec:
  installNamespace: open-cluster-management-agent-addon
  configs:
  # Secret with mTLS client certificate for application-logs output
  - resource: secrets
    name: managed-ocp-cluster-1-application-logs
    namespace: managed-ocp-cluster-1
  # ConfigMap with CloudWatch configuration for cluster-logs output
  - resource: configmaps
    name: managed-ocp-cluster-1-cluster-logs
    namespace: managed-ocp-cluster-1
```

For the `application-logs` log forwarding output the administrator needs to provide a Secret `managed-ocp-cluster-1-application-logs` with all required authentication information, e.g. TLS client Certificate:

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotation:
    logging.openshift.io/target-output-name: "application-logs"
  name: managed-ocp-cluster-1-application-logs
  namespace: managed-ocp-cluster-1
data:
  url: "https://loki-outside-this-cluster.com:3100"
  'tls.crt': "Base64 encoded TLS client certificate"
  'tls.key': "Base64 endoded TLS key"
  'ca-bundle.crt': "Base64 encoded Certificate Authority certificate"
```

For `cluster-logs` log forwarding output the administrator needs to provide a ConfigMap `managed-ocp-cluster-1-cluster-logs` with all output configuration, e.g. cloudwatch:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  annotation:
    logging.openshift.io/target-output-name: "cluster-logs"
  name: managed-ocp-cluster-1-cluster-logs
  namespace: managed-ocp-cluster-1
data:
  region: us-east-1
  groupPrefix: a-prefix
```

In turn the addon will compile a `ManifestWork` for the managed cluster `managed-ocp-cluster-1` as follows and pass it over the it's WorkAgentController:

```yaml
kind: ManifestWork
metadata:
  name: addon-logging-ocm-addon-deploy-0
  namespace: managed-cluster-1
spec:
  workload:
    manifests:
    - apiVersion: operators.coreos.com/v1alpha1
      kind: Subscription
      metadata:
        name: cluster-logging
        namespace: openshift-logging
      spec:
        channel: stable-5.8 # Pulled from the AddOnDeploymentConfig
        installPlanApproval: Automatic
        name: cluster-logging
        source: redhat-operators
        sourceNamespace: openshift-marketplace
        startingCSV: cluster-logging.v5.8.0
    - apiVersion: v1
      kind: Secret
      metadata:
        annotation:
          logging.openshift.io/target-output-name: "application-logs"
        name: managed-ocp-cluster-1-application-logs
        namespace: openshift-logging
      data:
        'tls.crt': "Base64 encoded TLS client certificate"
        'tls.key': "Base64 endoded TLS key"
        'ca-bundle.crt': "Base64 encoded Certificate Authority certificate"
    - apiVersion: logging.openshift.io/v1
      kind: ClusterLogForwarder
      metadata:
        name: instance
        namespace: open-cluster-management
      spec:
        outputs:
        - loki:
             labelKeys:
             - log_type
             - kubernetes.namespace_name
             - kubernetes.pod_name
             - openshift.cluster_id
          name: application-logs
          type: loki
          url: "https://loki-outside-this-cluster.com:3100" # Pulled from the referenced secret
          secret:
            name: managed-ocp-cluster-1-application-logs
        - name: cluster-logs
          type: cloudwatch
          cloudwatch:
            groupBy: logType
            region: us-east-1 # Pulled from the `managed-ocp-cluster-1-cluster-logs` configmap
            groupPrefix: a-prefix # Pulled from the `managed-ocp-cluster-1-cluster-logs` configmap
        pipelines:
        - inputRefs:
          - application
          name: application-logs
          outputRefs:
          - application-logs
        - inputRefs:
          - audit
          - infrastructure
          name: cluster-logs
          outputRefs:
          - cluster-logs
```

#### Multi Cluster Trace Collection and Forwarding

TBD

#### Variation and form factor considerations [optional]

TBD

### API Extensions

None

### Implementation Details/Notes/Constraints [optional]

TBD

#### Hypershift [optional]

TBD

### Drawbacks

TBD

## Design Details

### Open Questions [optional]

TBD

### Test Plan

TBD

### Graduation Criteria

TBD

#### Dev Preview -> Tech Preview

TBD

#### Tech Preview -> GA

TBD

#### Removing a deprecated feature

None

### Upgrade / Downgrade Strategy

None

### Version Skew Strategy

None

### Operational Aspects of API Extensions

TBD

#### Failure Modes

TBD

#### Support Procedures

TBD

## Implementation History

TBD

## Alternatives

TBD

## Infrastructure Needed [optional]

None

[ocm-addon-framework]:https://github.com/open-cluster-management-io/addon-framework
[ocp-cluster-logging-operator]:https://github.com/openshift/cluster-logging-operator
[opentelemetry-operator]:https://console-openshift-console.apps.ptsirakiaws2311285.devcluster.openshift.com/github.com/open-telemetry/opentelemetry-operator
[rhacm-multi-cluster-observability]:https://github.com/stolostron/multicluster-observability-operator/
