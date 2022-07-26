# Context

Deploy a fluent-bit helm chart that is able to get the logs and export them to a log analytics workspace in Azure. We need to make sure that the logs that we are getting just from an specific service.

# Requirements

## Helm

https://helm.sh/

### Helm values

https://helm.sh/docs/chart_template_guide/values_files/

## Kubernetes cluster

https://kubernetes.io/docs/tutorials/hello-minikube/

## Fluent-bit

https://docs.fluentbit.io/manual/

### Why fluent-bit instead of fluentd?

https://logz.io/blog/fluentd-vs-fluent-bit/

+better performance
+better adapted to k8s
-less plugins
+lightweigth
+no dependencies
-smaller community becuase is more recent

# Setup

## Install helm

https://helm.sh/docs/intro/install/

## Install minikube

https://minikube.sigs.k8s.io/docs/start/

## Understanding fluent-bit input, output, filter and parser

We will need to use [this](https://docs.fluentbit.io/manual/pipeline/inputs) to control the specific service that we want to export the logs. We will be able to put in *output* the log analytics workspace service to gather the logs like referred [here](https://docs.fluentbit.io/manual/pipeline/outputs/azure).

Currently we have two ways to get just the logs of the specific service.

* The recommended way of fluent-bit that should be just in the [filter kubernetes annotation](https://docs.fluentbit.io/manual/pipeline/filters/kubernetes#kubernetes-annotations) property where we create an annotation for each namespace specifying that we want to get the logs there.

* In the *input* property we can just [exclude_path](https://docs.fluentbit.io/manual/pipeline/inputs/tail) and just exclude the namespaces that we don't want to use like this `Exclude_Path /var/log/containers/*_kube-system_*.log, /var/log/containers/*_logs_*.log`

 and we will need to filter the *inputs* to avoid getting the logs of all pods with [exclude_path](https://docs.fluentbit.io/manual/pipeline/inputs/tail) property.

## Manipulate helm file values

This is the most important part for us to change in the values.yaml of the fluent-bit helm chart. In this part we can change the properties to allow the use case that we want.

```yaml
## https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/configuration-file
config:
  service: |
    [SERVICE]
        Daemon Off
        Flush {{ .Values.flush }}
        Log_Level {{ .Values.logLevel }}
        Parsers_File parsers.conf
        Parsers_File custom_parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port {{ .Values.metricsPort }}
        Health_Check On
  ## https://docs.fluentbit.io/manual/pipeline/inputs
  inputs: |
    [INPUT]
        Name tail
        Tag  kube.*
        Path /var/log/containers/*.log
        Exclude_Path /var/log/containers/*_kube-system_*.log, /var/log/containers/*_logs_*.log
        Parser docker
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10
  ## https://docs.fluentbit.io/manual/pipeline/filters
  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log Off
        Keep_Log On
        K8S-Logging.Exclude On
  ## https://docs.fluentbit.io/manual/pipeline/outputs
  outputs: |
    [OUTPUT]
        Name stdout
        Match *
  ## https://docs.fluentbit.io/manual/pipeline/parsers
  customParsers: |
    [PARSER]
        Name docker
        Format json
        Time_Keep Off
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
```

# Usage

##  Helm chart of fluent-bit

https://github.com/fluent/helm-charts/tree/main/charts/fluent-bit

## Run and install helm chart fluent-bit

Install the chart in the namespace *logs*

`helm install fluent-bit fluent/fluent-bit -n logs -f values.yaml`

## Uninstall helm chart fluent-bit

Uninstall the chart in the namespace *logs*

`helm uninstall fluent-bit fluent/fluent-bit -n logs -f values.yaml`