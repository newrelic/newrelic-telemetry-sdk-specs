## Telemetry
A collection of measurements or other data at a remote system or inaccessible points. Telemetry in the New Relic context refers to metrics, events, logs or traces, or MELT. See official documentation [add link]

## Harvester
The harvester is a higher level construct that exposes methods to record telemetry, batching the telemetry, aggregating the telemetry, and sending the data to New Relic at an interval. 

## Batch
A batch is a collection of telemetry data of the same type, e.g. a batch of metrics, a batch of spans. The telemetry SDKs sends batches of telemetry data to New Relic. A batch can also have attributes that are common for all contained telemetry data.

## Metric
A metric, at a conceptual level, is a measurement. The measurement is identified by a `name` and `type` and `attribute keys + values`. 
### Aggregation
Multiple Metric data can be combined into a single metric by calculating the combined measurements. The type of calculation varies by metric type. 
#### Gauge Metric Type
Represents the value of something at that moment in time. The value can go up or down over time. No aggregation occurs with the gauge metric type
#### Count Metric Type
Measures the occurrences of an event. When count metrics are aggregated, the new resulting count metric has a count that is the sum of the input count metrics.
#### Summary Metric Type
Used for reporting the count, average, sum, min, and max values of discrete events. When summary metrics are aggregated, each summary value is recalculated.

## Trace
A trace or a distributed trace is a causal chain of events between different components.
### Span
An individual unit of work done in a distributed system.





