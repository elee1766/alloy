---
canonical: https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.remote.queue/
aliases:
  - ../prometheus.remote.queue/ # /docs/alloy/latest/reference/components/prometheus.remote.queue/
description: Learn about prometheus.remote.queue
title: prometheus.remote.queue
---

# prometheus.remote.queue

`prometheus.remote.queue` collects metrics sent from other components into a
Write-Ahead Log (WAL) and forwards them over the network to a series of
user-supplied endpoints. Metrics are sent over the network using the
[Prometheus Remote Write protocol][remote_write-spec].

Multiple `prometheus.remote.queue` components can be specified by giving them
different labels.

[remote_write-spec]: https://docs.google.com/document/d/1LPhVRSFkGNSuU1fBd81ulhsCPR4hkSZyyBj1SZ8fWOM/edit

## Usage

```alloy
prometheus.remote.queue "LABEL" {
  endpoint {
    url = REMOTE_WRITE_URL

    ...
  }

  ...
}
```

## Arguments

The following arguments are supported:

Name | Type | Description | Default | Required
---- | ---- | ----------- | ------- | --------
`ttl` | `time` | `duration` | How long the timestamp of a signal is valid for, before the signal is discarded. | `2h` | no
`max_signals_to_batch` | `uint` | The maximum number of signals before they are batched to disk. | `10,000` | no
`batch_frequency` | `duration` | How often to batch signals to disk if `max_signals_to_batch` is not reached. | no


## Blocks

The following blocks are supported inside the definition of
`prometheus.remote.queue`:

Hierarchy | Block | Description | Required
--------- | ----- | ----------- | --------
endpoint | [endpoint][] | Location to send metrics to. | no
endpoint > basic_auth | [basic_auth][] | Configure basic_auth for authenticating to the endpoint. | no

The `>` symbol indicates deeper levels of nesting. For example, `endpoint >
basic_auth` refers to a `basic_auth` block defined inside an
`endpoint` block.

[endpoint]: #endpoint-block
[basic_auth]: #basic_auth-block

### endpoint block

The `endpoint` block describes a single location to send metrics to. Multiple
`endpoint` blocks can be provided to send metrics to multiple locations.

The following arguments are supported:

Name | Type | Description | Default | Required
---- | ---- | ----------- | ------- | --------
`url` | `string` | Full URL to send metrics to. | | yes
`name` | `string` | Optional name to identify the endpoint in metrics. | | no
`write_timeout` | `duration` | Timeout for requests made to the URL. | `"30s"` | no
`retry_backoff` | `duration` | How often to wait between retries. | `1s` | no
`max_retry_backoff_attempts` | Maximum number of retries before dropping the batch. | `1s` | no
`batch_count` | `uint` | How many series to queue in each queue. | `1,000` | no
`flush_frequency` | `duration` | How often to wait until sending if `batch_count` is not trigger. | `1s` | no
`queue_count` | `uint` | How many concurrent batches to write. | 10 | no
`external_labels` | `map(string)` | Labels to add to metrics sent over the network. | | no

### basic_auth block

{{< docs/shared lookup="reference/components/basic-auth-block.md" source="alloy" version="<ALLOY_VERSION>" >}}


## Exported fields

The following fields are exported and can be referenced by other components:

Name | Type | Description
---- | ---- | -----------
`receiver` | `MetricsReceiver` | A value which other components can use to send metrics to.

## Component health

`prometheus.remote.queue` is only reported as unhealthy if given an invalid
configuration. In those cases, exported fields are kept at their last healthy
values.

## Debug information

`prometheus.remote_write` does not expose any component-specific debug
information.

## Debug metrics

The below metrics are provided for backwards compatibility, they behave generally the same but there are likely
edge cases where they differ.

* `prometheus_remote_write_wal_storage_created_series_total` (counter): Total number of created
  series appended to the WAL.
* `prometheus_remote_write_wal_storage_removed_series_total` (counter): Total number of series
  removed from the WAL.
* `prometheus_remote_write_wal_samples_appended_total` (counter): Total number of samples
  appended to the WAL.
* `prometheus_remote_write_wal_exemplars_appended_total` (counter): Total number of exemplars
  appended to the WAL.
* `prometheus_remote_storage_samples_total` (counter): Total number of samples
  sent to remote storage.
* `prometheus_remote_storage_exemplars_total` (counter): Total number of
  exemplars sent to remote storage.
* `prometheus_remote_storage_metadata_total` (counter): Total number of
  metadata entries sent to remote storage.
* `prometheus_remote_storage_samples_failed_total` (counter): Total number of
  samples that failed to send to remote storage due to non-recoverable errors.
* `prometheus_remote_storage_exemplars_failed_total` (counter): Total number of
  exemplars that failed to send to remote storage due to non-recoverable errors.
* `prometheus_remote_storage_metadata_failed_total` (counter): Total number of
  metadata entries that failed to send to remote storage due to
  non-recoverable errors.
* `prometheus_remote_storage_samples_retries_total` (counter): Total number of
  samples that failed to send to remote storage but were retried due to
  recoverable errors.
* `prometheus_remote_storage_exemplars_retried_total` (counter): Total number of
  exemplars that failed to send to remote storage but were retried due to
  recoverable errors.
* `prometheus_remote_storage_metadata_retried_total` (counter): Total number of
  metadata entries that failed to send to remote storage but were retried due
  to recoverable errors.
* `prometheus_remote_storage_samples_dropped_total` (counter): Total number of
  samples which were dropped after being read from the WAL before being sent to
  remote_write because of an unknown reference ID.
* `prometheus_remote_storage_exemplars_dropped_total` (counter): Total number
  of exemplars which were dropped after being read from the WAL before being
  sent to remote_write because of an unknown reference ID.
* `prometheus_remote_storage_enqueue_retries_total` (counter): Total number of
  times enqueue has failed because a shard's queue was full.
* `prometheus_remote_storage_sent_batch_duration_seconds` (histogram): Duration
  of send calls to remote storage.
* `prometheus_remote_storage_queue_highest_sent_timestamp_seconds` (gauge):
  Unix timestamp of the latest WAL sample successfully sent by a queue.
* `prometheus_remote_storage_samples_pending` (gauge): The number of samples
  pending in shards to be sent to remote storage.
* `prometheus_remote_storage_exemplars_pending` (gauge): The number of
  exemplars pending in shards to be sent to remote storage.
* `prometheus_remote_storage_samples_in_total` (counter): Samples read into
  remote storage.
* `prometheus_remote_storage_exemplars_in_total` (counter): Exemplars read into
  remote storage.

TODO document new metrics.

## Examples

The following examples show you how to create `prometheus.remote_write` components that send metrics to different destinations.

### Send metrics to a local Mimir instance

You can create a `prometheus.remote.queue` component that sends your metrics to a local Mimir instance:

```alloy
prometheus.remote.queue "staging" {
  // Send metrics to a locally running Mimir.
  endpoint "mimir" {
    url = "http://mimir:9009/api/v1/push"

    basic_auth {
      username = "example-user"
      password = "example-password"
    }
  }
}

// Configure a prometheus.scrape component to send metrics to
// prometheus.remote_write component.
prometheus.scrape "demo" {
  targets = [
    // Collect metrics from the default HTTP listen address.
    {"__address__" = "127.0.0.1:12345"},
  ]
  forward_to = [prometheus.remote.queue.staging.receiver]
}

```

## TODO Metadata settings

## Technical details

`prometheus.remote.queue` uses [snappy][] for compression.
`prometheus.remote.queue` sends native histograms by default.
Any labels that start with `__` will be removed before sending to the endpoint.

### Data retention

Data is written to disk in blocks utilizing [snappy][] compression. These blocks are read on startup and resent if they are still within the TTL. 
Any data that has not been written to disk, or that is in the network queues is lost if Alloy is restarted.

### Retries

Network errors will be retried. 429 errors will be retried. 5XX errors will retry. Any other non-2XX return codes will not be tried. 

### Memory

`prometheus.remote.queue` is meant to be memory efficient. By adjusting the `max_signals_to_batch`, `queue_count`, and `batch_size` the amount of memory
can be controlled. A higher `max_signals_to_batch` allows for more efficient disk compression. A higher `queue_count` allows more concurrent writes and `batch_size`
allows more data sent at one time. This can allow greater throughput, at the cost of more memory on both Alloy and the endpoint. The defaults are good for most 
common usages. 

## Compatible components

`prometheus.remote.queue` has exports that can be consumed by the following components:

- Components that consume [Prometheus `MetricsReceiver`](../../../compatibility/#prometheus-metricsreceiver-consumers)

{{< admonition type="note" >}}
Connecting some components may not be sensible or components may require further configuration to make the connection work correctly.
Refer to the linked documentation for more details.
{{< /admonition >}}

<!-- END GENERATED COMPATIBLE COMPONENTS -->

[snappy]: https://en.wikipedia.org/wiki/Snappy_(compression)
[WAL block]: #wal-block
[Stop]: ../../../../set-up/run/
[run]: ../../../cli/run/