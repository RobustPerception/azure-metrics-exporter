# Azure-metrics-exporter

**Work in progress**

Azure metrics exporter for [Prometheus.](https://prometheus.io)

Allows for the exporting of metrics from Azure applications using the [Azure monitor API.](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-rest-api-walkthrough)

# Rate limits

Note that Azure imposes an [API read limit of 15,000 requests per hour](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-request-limits) so the number of metrics you're querying for should be proportional to your scrape interval.

# Retrieving Metric definitions

In order to get all the metric definitions for the resources specified in your configuration file, run the following:

`./Azure-metrics-exporter --list.definitions`

This will print your resource id's application/service name along with a list of each of the available metric definitions that you can query for for that resource.

# Example azure-metrics-exporter config
`active_directory_authority_url` is AzureAD url for getting access token.
`resource_manager_url` is Azure API management url.

These parameters are in the configuration because of the differences between national clouds (e.g. AzureChina cloud has another endpoints).
You can find endpoints for national clouds [here](http://www.azurespeed.com/Information/AzureEnvironments)

`azure_resource_id` and `subscription_id` can be found under properties in the Azure portal for your application/service.

`azure_resource_id`  should start with `/resourceGroups...` (`/subscriptions/xxxxxxxx-xxxx-xxxx-xxx-xxxxxxxxx` must be removed from the begining of `azure_resource_id` property value)

`tenant_id` is found under `Azure Active Directory > Properties` and is listed as `Directory ID`.

The `client_id` and `client_secret` are obtained by registering an application under 'Azure Active Directory'.

`client_id` is the `application_id` of your application and the `client_secret` is generated by selecting your application/service under Azure Active Directory, selecting 'keys', and generating a new key.

Then for the resource group your application is apart of, create a new IAM reader role for the created app under 'Azure Active Directory'.

```
active_directory_authority_url: "https://login.microsoftonline.com/"
resource_manager_url: "https://management.azure.com/"
credentials:
  subscription_id: <secret>
  client_id: <secret>
  client_secret: <secret>
  tenant_id: <secret>

targets:
  - resource: "azure_resource_id"
    metrics:
    - name: "BytesReceived"
    - name: "BytesSent"
  - resource: "azure_resource_id"
    aggregations:
    - Minimum
    - Maximum
    - Average
    metrics:
    - name: "Http2xx"
    - name: "Http5xx"

resource_groups:
  - resource_group: "webapps"
    resource_types:
    - "Microsoft.Compute/virtualMachines"
    resource_name_include_re:
    - "testvm.*"
    resource_name_exclude_re:
    - "testvm12"
    metrics:
    - name: "CPU Credits Consumed"

```

By default, all aggregations are returned (`Total`, `Maximum`, `Average`, `Minimum`). It can be overridden per resource.

# Example Prometheus config

```
global:
  scrape_interval:     60s # Set a high scrape_interval either globally or per-job to avoid hitting Azure Monitor API limits.

scrape_configs:
  - job_name: azure
    static_configs:
      - targets: ['localhost:9276']
```

# Resource group filtering

Resources in a resource group can be filtered using the the following keys:

`resource_types`:
List of resource types to include (corresponds to the `Resource type` column in the Azure portal).

`resource_name_include_re`:
List of regexps that is matched against the resource name.
Metrics of all matched resources are exported (defaults to include all)

`resource_name_exclude_re`:
List of regexps that is matched against the resource name.
Metrics of all matched resources are ignored (defaults to exclude none)
Excludes take precedence over the include filter.
