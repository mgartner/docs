---
title: Export Metrics From a CockroachDB Dedicated Cluster
summary: Export Metrics From a CockroachDB Dedicated Cluster
toc: true
docs_area: manage
cloud: true
---

CockroachDB {{ site.data.products.dedicated }} users can use the [Cloud API]({% link cockroachcloud/cloud-api.md %}) to configure metrics export to [AWS CloudWatch](https://aws.amazon.com/cloudwatch/) or [Datadog](https://www.datadoghq.com/). Once the export is configured, metrics will flow from all nodes in all regions of your CockroachDB {{ site.data.products.dedicated }} cluster to your chosen cloud metrics sink.

{{site.data.alerts.callout_success}}
CockroachDB {{ site.data.products.dedicated }} clusters use Cloud Console instead of DB Console, and DB Console is disabled. To export metrics from a CockroachDB {{ site.data.products.core }} cluster, refer to [Monitoring and Alerting](https://www.cockroachlabs.com/docs/{{site.current_cloud_version}}/monitoring-and-alerting) instead of this page.
{{site.data.alerts.end}}

Exporting metrics to AWS CloudWatch is only available on CockroachDB {{ site.data.products.dedicated }} clusters which are hosted on AWS. Metrics export to Datadog is supported on all CockroachDB {{ site.data.products.dedicated }} clusters.

{{site.data.alerts.callout_info}}
{% include_cached feature-phases/preview.md %}
{{site.data.alerts.end}}

<a id="the-metricexport-endpoint"></a>

## `metricexport` endpoint

To configure and manage metrics export for your CockroachDB {{ site.data.products.dedicated }} cluster, use the `metricexport` endpoint appropriate for your desired cloud metrics sink:

Cloud metrics sink | Metrics export endpoint
------------------ | ----------------------------------------------------
AWS Cloudwatch     | `https://cockroachlabs.cloud/api/v1/clusters/{your_cluster_id}/metricexport/cloudwatch`
Datadog            | `https://cockroachlabs.cloud/api/v1/clusters/{your_cluster_id}/metricexport/datadog`

Access to the `metricexport` endpoints requires a valid CockroachDB {{ site.data.products.cloud }} [service account]({% link cockroachcloud/managing-access.md %}#manage-service-accounts) with the appropriate permissions (`admin` privilege or Cluster Admin role).

The following methods are available for use with the `metricexport` endpoints, and require the listed service account permissions:

Method | Required permissions | Description
--- | --- | ---
`GET` | `ADMIN`, `EDIT`, or `READ` | Returns the current status of the metrics export configuration.
`POST` | `ADMIN` or `EDIT` | Enables metrics export, or updates an existing metrics export configuration.
`DELETE` | `ADMIN` | Disables metrics export, halting all metrics export to AWS CloudWatch or Datadog.

See [Service accounts]({% link cockroachcloud/managing-access.md %}#manage-service-accounts) for instructions on configuring a service account with these required permissions.

## Enable metrics export

<div class="filters clearfix">
  <button class="filter-button" data-scope="aws-metrics-export">AWS CloudWatch</button>
  <button class="filter-button" data-scope="datadog-metrics-export">Datadog</button>
</div>

<section class="filter-content" markdown="1" data-scope="aws-metrics-export">

Exporting metrics to AWS CloudWatch is only available on CockroachDB {{ site.data.products.dedicated }} clusters which are hosted on AWS. If your CockroachDB {{ site.data.products.dedicated }} cluster is hosted on GCP or Azure, you can [export metrics to Datadog](export-metrics.html?filters=datadog-metrics-export) instead.

{{site.data.alerts.callout_info}}
Enabling metrics export will send around 250 metrics per node to AWS CloudWatch. Review the [AWS CloudWatch documentation](https://aws.amazon.com/cloudwatch/pricing/) to gauge how this adds to your AWS CloudWatch spend.
{{site.data.alerts.end}}

Perform the following steps to enable metrics export from your CockroachDB {{ site.data.products.dedicated }} cluster to AWS CloudWatch.

1. Create the desired target AWS CloudWatch log group by following the [Create a log group in CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html#Create-Log-Group) instructions. If you already have a log group created, you may skip this step. When your CockroachDB {{ site.data.products.dedicated }} cluster emits metrics to AWS CloudWatch, they are written to this log group.

1. Find your CockroachDB {{ site.data.products.dedicated }} cluster ID:

	1. Visit the CockroachDB {{ site.data.products.cloud }} console [cluster page](https://cockroachlabs.cloud/clusters).
	1. Click on the name of your cluster.
	1. Find your cluster ID in the URL of the single cluster overview page: `https://cockroachlabs.cloud/cluster/{your_cluster_id}/overview`. It should resemble `f78b7feb-b6cf-4396-9d7f-494982d7d81e`.

1. Determine your cluster's cloud provider account ID. This command uses the third-party JSON parsing tool [`jq`](https://stedolan.github.io/jq/download/) to isolate just the needed `account_id` field:

    {% include_cached copy-clipboard.html %}
    ~~~shell
    curl --request GET \
      --url https://cockroachlabs.cloud/api/v1/clusters/{your_cluster_id} \
      --header 'Authorization: Bearer {secret_key}' | jq .account_id
    ~~~

    Where:
    - `{your_cluster_id}` is the cluster ID of your CockroachDB {{ site.data.products.dedicated }} cluster as determined in step 2.
    - `{secret_key}` is your CockroachDB {{ site.data.products.dedicated }} API key. See [API Access]({% link cockroachcloud/managing-access.md %}) for more details.

1.  Create a cross-account IAM role in your AWS account:

	1. In the AWS console, visit the [IAM page](https://console.aws.amazon.com/iam/).
	1. Select **Roles** and click **Create role**.
	1. For **Trusted entity type**, select **AWS account**.
	1. Choose **Another AWS account**.
	1. For **Account ID**, provide the CockroachDB {{ site.data.products.dedicated }} cloud provider account ID from step 3.
	1. Finish creating the IAM role with a suitable name. These instructions will use the role name `CockroachCloudMetricsExportRole`. You do not need to add any permissions.

	{{site.data.alerts.callout_info}}
	You will need the Amazon Resource Name (ARN) for your cross-account IAM role later in this procedure.
	{{site.data.alerts.end}}

1. Select the new role, and create a new policy for this role. These instructions will use the policy name `CockroachCloudMetricsExportPolicy`.

1. Select the new policy, and paste the following into the **Permissions** tab, with the **{} JSON** option selected:

    {% include_cached copy-clipboard.html %}
    ~~~
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:DescribeLogGroups",
                    "logs:DescribeLogStreams",
                    "logs:PutRetentionPolicy",
                    "logs:PutLogEvents"
                ],
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:logs:*:{your_aws_acct_id}:log-group:{log_group_name}",
                    "arn:aws:logs:*:{your_aws_acct_id}:log-group:{log_group_name}:log-stream:*"
                ]
            }
        ]
    }
    ~~~

    Where:
    - `{your_aws_acct_id}` is the AWS Account ID of the AWS account where you created the `CockroachCloudMetricsExportRole` role, **not** the cloud provider account ID of your CockroachDB {{ site.data.products.dedicated }} cluster from step 3. You can find your AWS Account ID on the AWS [IAM page](https://console.aws.amazon.com/iam/).
    - `{log_group_name}` is the target AWS CloudWatch log group you created in step 1.

    This defines the set of permissions that the CockroachDB {{ site.data.products.dedicated }} metrics export feature requires to be able to write metrics to CloudWatch.

1. Copy the [Amazon Resource Name (ARN)](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html) of the `CockroachCloudMetricsExportRole` role found under **Summary**, which is needed for the next step.

1. Issue the following Cloud API command to enable metrics export for your CockroachDB {{ site.data.products.dedicated }} cluster:

    {% include_cached copy-clipboard.html %}
    ~~~shell
    curl --request POST \
      --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/metricexport/cloudwatch \
      --header "Authorization: Bearer {secret_key}" \
      --data '{"target_region": "{aws_region}", "role_arn": "arn:aws:iam::{role_arn}:role/CockroachCloudMetricsExportRole", "log_group_name": "{log_group_name}"}'
    ~~~

    Where:
    - `{cluster_id}` is your CockroachDB {{ site.data.products.dedicated }} cluster ID as determined in step 2.
    - `{secret_key}` is your CockroachDB {{ site.data.products.dedicated }} API key. See [API Access]({% link cockroachcloud/managing-access.md %}) for instructions on generating this key.
    - `{aws_region}` is your AWS region, like `us-east-1`.
    - `{role_arn}` is the ARN for the `CockroachCloudMetricsExportRole` role you copied in step 7. If you used a different role name there, be sure to use your role name in place of `CockroachCloudMetricsExportRole` in the above command.
    - `{log_group_name}` is the target AWS CloudWatch log group you created in step 1. This **must** be the same group name you provided in step 6.

    Specifying an AWS region (to `{aws_region}`) that you do not have a cluster in, or a region that only partially covers your cluster's nodes will result in missing metrics.

1. Depending on the size of your cluster and how many regions it spans, the configuration may take a moment. You can monitor the ongoing status of the configuration using the following Cloud API command:

    {% include_cached copy-clipboard.html %}
    ~~~shell
    curl --request GET \
      --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/metricexport/cloudwatch \
      --header "Authorization: Bearer {secret_key}"
    ~~~

    Run the command periodically until the command returns a status of `ENABLED`, at which point the configuration across all nodes is complete, and metrics will begin appearing in CloudWatch under the log group you created in step 1. Since the configuration is applied to cluster nodes in a rolling fashion, you may see some metrics appear even before the `GET` command returns an `ENABLED` status.

1. Once metrics export has been enabled, you can access metrics from your CockroachDB {{ site.data.products.dedicated }} cluster directly in [AWS CloudWatch](https://console.aws.amazon.com/cloudwatch/home).

</section>

<section class="filter-content" markdown="1" data-scope="datadog-metrics-export">

To enable metrics export for your CockroachDB {{ site.data.products.dedicated }} cluster to Datadog, you can either enable the Datadog integration in your CockroachDB {{ site.data.products.dedicated }} Cloud Console, or on the command line via the [Cloud API]({% link cockroachcloud/cloud-api.md %}):

- To enable metrics export to Datadog using the Cloud Console, follow the [Monitor CockroachDB {{ site.data.products.dedicated }} with Datadog]({% link cockroachcloud/tools-page.md %}#monitor-cockroachdb-dedicated-with-datadog) instructions.
- To enable metrics export to Datadog using the [Cloud API]({% link cockroachcloud/cloud-api.md %}), follow the steps below.

1. Find your CockroachDB {{ site.data.products.dedicated }} cluster ID:

	1. Visit the CockroachDB {{ site.data.products.cloud }} console [cluster page](https://cockroachlabs.cloud/clusters).
	1. Click on the name of your cluster.
	1. Find your cluster ID in the URL of the single cluster overview page: `https://cockroachlabs.cloud/cluster/{your_cluster_id}/overview`. It should resemble `f78b7feb-b6cf-4396-9d7f-494982d7d81e`.

1. Determine the [Datadog API key](https://docs.datadoghq.com/account_management/api-app-keys/) you'd like to use. If you don't already have one, follow the steps to [add a new Datadog API key](https://docs.datadoghq.com/account_management/api-app-keys/#add-an-api-key-or-client-token).

1. Issue the following Cloud API command to enable metrics export for your CockroachDB {{ site.data.products.dedicated }} cluster:

    {% include_cached copy-clipboard.html %}
    ~~~shell
    curl --request POST \
      --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/metricexport/datadog \
      --header "Authorization: Bearer {secret_key}" \
      --data '{"site": "{datadog_site}", "api_key": "{datadog_api_key}"}'
    ~~~

    Where:
    - `{cluster_id}` is your CockroachDB {{ site.data.products.dedicated }} cluster ID as determined in step 1, resembling `f78b7feb-b6cf-4396-9d7f-494982d7d81e`.
    - `{secret_key}` is your CockroachDB {{ site.data.products.dedicated }} API key. See [API Access]({% link cockroachcloud/managing-access.md %}) for instructions on generating this key.
    - `{datadog_site}` is your Datadog site. Valid sites are: `US1`, `US3`, `US5`, `US1_GOV`, and `EU1`.
    - `{datadog_api_key}` is the Datadog API key determined in step 2.

1. Depending on the size of your cluster and how many regions it spans, the configuration may take a moment. You can monitor the ongoing status of the configuration using the following Cloud API command:

    {% include_cached copy-clipboard.html %}
    ~~~shell
    curl --request GET \
      --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/metricexport/datadog \
      --header "Authorization: Bearer {secret_key}"
    ~~~

    Run the command periodically until the command returns a status of `ENABLED`, at which point the configuration across all nodes is complete, and metrics will begin appearing in the Datadog interface. Since the configuration is applied to cluster nodes in a rolling fashion, you may see some metrics appear even before the `GET` command returns an `ENABLED` status.

1. Once metrics export has been enabled, you can access metrics from your CockroachDB {{ site.data.products.dedicated }} cluster directly in Datadog's [Metrics Explorer](https://docs.datadoghq.com/metrics/explorer/), or via Datadog's [notebook](https://docs.datadoghq.com/notebooks/) or [dashboard](https://docs.datadoghq.com/dashboards/) features.

1. Review [enable percentiles for selected metrics]({% link cockroachcloud/tools-page.md %}#enable-percentiles-for-selected-metrics). Configure metrics as necessary.

{{site.data.alerts.callout_info}}
A subset of CockroachDB metrics require that you explicitly [enable percentiles]({% link cockroachcloud/tools-page.md %}#enable-percentiles-for-selected-metrics) for them in the Datadog interface. Graphs that display data for these metrics will fail to render properly otherwise.
{{site.data.alerts.end}}

</section>

## Monitor the status of a metrics export configuration

<div class="filters clearfix">
  <button class="filter-button" data-scope="aws-metrics-export">AWS CloudWatch</button>
  <button class="filter-button" data-scope="datadog-metrics-export">Datadog</button>
</div>

<section class="filter-content" markdown="1" data-scope="aws-metrics-export">

To check the status of an existing AWS Cloudwatch metrics export configuration, use the following Cloud API command:

{% include_cached copy-clipboard.html %}
~~~shell
curl --request GET \
  --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/metricexport/cloudwatch \
  --header "Authorization: Bearer {secret_key}"
~~~

Where:

- `{cluster_id}` is your CockroachDB {{ site.data.products.dedicated }} cluster's cluster ID, which can be found in the URL of your [Cloud Console](https://cockroachlabs.cloud/clusters/) for the specific cluster you wish to configure, resembling `f78b7feb-b6cf-4396-9d7f-494982d7d81e`.
- `{secret_key}` is your CockroachDB {{ site.data.products.dedicated }} API key. See [API Access]({% link cockroachcloud/managing-access.md %}) for instructions on generating this key.

</section>

<section class="filter-content" markdown="1" data-scope="datadog-metrics-export">

To check the status of an existing Datadog metrics export configuration, use the following Cloud API command:

{% include_cached copy-clipboard.html %}
~~~shell
curl --request GET \
  --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/metricexport/datadog \
  --header "Authorization: Bearer {secret_key}"
~~~

Where:

- `{cluster_id}` is your CockroachDB {{ site.data.products.dedicated }} cluster's cluster ID, which can be found in the URL of your [Cloud Console](https://cockroachlabs.cloud/clusters/) for the specific cluster you wish to configure, resembling `f78b7feb-b6cf-4396-9d7f-494982d7d81e`.
- `{secret_key}` is your CockroachDB {{ site.data.products.dedicated }} API key. See [API Access]({% link cockroachcloud/managing-access.md %}) for instructions on generating this key.

</section>

## Update an existing metrics export configuration

To update an existing CockroachDB {{ site.data.products.dedicated }} metrics export configuration, make any necessary changes to your cloud provider configuration (e.g., AWS CloudWatch or Datadog), then issue the same `POST` Cloud API command as shown in the [Enable metrics export](#enable-metrics-export) instructions for your cloud provider with the desired updated configuration. Follow the [Monitor the status of a metrics export configuration](#monitor-the-status-of-a-metrics-export-configuration) instructions to ensure the update completes successfully.

## Disable metrics export

<div class="filters clearfix">
  <button class="filter-button" data-scope="aws-metrics-export">AWS CloudWatch</button>
  <button class="filter-button" data-scope="datadog-metrics-export">Datadog</button>
</div>

<section class="filter-content" markdown="1" data-scope="aws-metrics-export">

To disable an existing AWS Cloudwatch metrics export configuration, and stop sending metrics to Cloudwatch, use the following Cloud API command:

{% include_cached copy-clipboard.html %}
~~~shell
curl --request DELETE \
  --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/metricexport/cloudwatch \
  --header "Authorization: Bearer {secret_key}"
~~~

Where:

- `{cluster_id}` is your CockroachDB {{ site.data.products.dedicated }} cluster's cluster ID, which can be found in the URL of your [Cloud Console](https://cockroachlabs.cloud/clusters/) for the specific cluster you wish to configure, resembling `f78b7feb-b6cf-4396-9d7f-494982d7d81e`.
- `{secret_key}` is your CockroachDB {{ site.data.products.dedicated }} API key. See [API Access]({% link cockroachcloud/managing-access.md %}) for instructions on generating this key.

</section>

<section class="filter-content" markdown="1" data-scope="datadog-metrics-export">

To disable an existing Datadog metrics export configuration, and stop sending metrics to Datadog, use the following Cloud API command:

{% include_cached copy-clipboard.html %}
~~~shell
curl --request DELETE \
  --url https://cockroachlabs.cloud/api/v1/clusters/{cluster_id}/metricexport/datadog \
  --header "Authorization: Bearer {secret_key}"
~~~

Where:

- `{cluster_id}` is your CockroachDB {{ site.data.products.dedicated }} cluster's cluster ID, which can be found in the URL of your [Cloud Console](https://cockroachlabs.cloud/clusters/) for the specific cluster you wish to configure, resembling `f78b7feb-b6cf-4396-9d7f-494982d7d81e`.
- `{secret_key}` is your CockroachDB {{ site.data.products.dedicated }} API key. See [API Access]({% link cockroachcloud/managing-access.md %}) for instructions on generating this key.

</section>

## Limitations

- Metrics export to AWS CloudWatch is only available on CockroachDB {{ site.data.products.dedicated }} clusters which are hosted on AWS. If your CockroachDB {{ site.data.products.dedicated }} cluster is hosted on GCP or Azure, you can [export metrics to Datadog](export-metrics.html?filters=datadog-metrics-export) instead.
- AWS CloudWatch does not currently support histograms. Any histogram-type metrics emitted from your CockroachDB {{ site.data.products.dedicated }} cluster are dropped by CloudWatch. See [Prometheus metric type conversion](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights-Prometheus-metrics-conversion.html) for more information, and [Logging dropped Prometheus metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights-Prometheus-troubleshooting-EKS.html#ContainerInsights-Prometheus-troubleshooting-droppedmetrics) for instructions on tracking dropped histogram metrics in CloudWatch.

## Troubleshooting

### AWS CloudWatch

Be sure you are providing **your own** AWS Account ID as shown on the AWS [IAM page](https://console.aws.amazon.com/iam/) to step 6, **not** the AWS cloud provider account ID as returned from step 3.

If you are using an existing AWS role, or are otherwise using a role name different from the example name used in this tutorial, be sure to use your own role name in step 8 in place of `CockroachCloudMetricsExportRole`.

Your CockroachDB {{ site.data.products.dedicated }} cluster must be running on AWS (not GCP or Azure) to make use of metrics export to AWS CloudWatch. If your CockroachDB {{ site.data.products.dedicated }} cluster is hosted on GCP or Azure, you can [export metrics to Datadog](export-metrics.html?filters=datadog-metrics-export) instead.

## See Also

- [Differences in Metrics between Third-Party Monitoring Integrations and DB Console]({% link {{site.current_cloud_version}}/differences-in-metrics-between-third-party-monitoring-integrations-and-db-console.md %})