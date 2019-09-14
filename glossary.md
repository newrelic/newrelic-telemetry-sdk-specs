The main [New Relic glossary](https://docs.newrelic.com/docs/using-new-relic/welcome-new-relic/get-started/glossary) contains the list of terminology commonly encountered by New Relic users. This glossary containts the list of terminology particularly importand for developers trying to understand how the Telemetry SDKs work.

## Telemetry
A collection of measurements or other data at a remote system or inaccessible points. Telemetry in the Telemetry SDKs context currently refer to [metrics](#metric) or [traces](#trace).

## Harvester
The harvester is a higher level construct that exposes methods to record telemetry, batching the telemetry, aggregating the telemetry, and sending the data to New Relic at an interval. 

## Batch
A batch is a collection of telemetry data of the same type, e.g. a batch of metrics, a batch of spans. The telemetry SDKs sends batches of telemetry data to New Relic. A batch can also have attributes that are common for all contained telemetry data.

## Metric
A metric, at a conceptual level, is a measurement. The measurement is identified by a `name` and `type` and `attribute keys + values`. The [New Relic Metric API](https://docs.newrelic.com/docs/introduction-new-relic-metric-api) accepts batches of metrics sent by the Telemetry SDKs.
### Aggregation
Multiple Metric data can be combined into a single metric by calculating the combined measurements. The type of calculation varies by metric type. 
#### Gauge Metric Type
Represents the value of something at that moment in time. The value can go up or down over time. No aggregation occurs with the gauge metric type
#### Count Metric Type
Measures the occurrences of an event. When count metrics are aggregated, the new resulting count metric has a count that is the sum of the input count metrics.
#### Summary Metric Type
Used for reporting the count, average, sum, min, and max values of discrete events. When summary metrics are aggregated, each summary value is recalculated.

## Trace
A trace or a distributed trace is a causal chain of events between different components. It is composed of a collection of [spans](https://docs.newrelic.com/docs/using-new-relic/welcome-new-relic/get-started/glossary#span) sharing the same traceId. The [New Relic Trace API](https://docs.newrelic.com/docs/apm/distributed-tracing/trace-api/introduction-new-relic-trace-api) accepts batches of spans sent by the Telemetry SDKs.
### Span
An individual unit of work done in a distributed system.





