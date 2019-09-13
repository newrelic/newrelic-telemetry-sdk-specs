# Telemetry SDK Capabilities

The Telemetry SDK is capable of sending multiple types of telemetry.

## Metrics

The Telemetry SDK implements the New Relic Metric Data Model.  For detailed information
about the Metric model, please see the [public docs](link_to_documentation).

The SDK must support two primary use cases:
1. The SDK must provide a representation of each of New Relic's supported metric types.
   These representations may be immutable data structures, or they may be mutable objects,
   depending on the idioms of the language.  It must be possible to construct them with
   pre-computed values, and the SDK must be capable of serializing them into the for
   transport to New Relic.
2. The SDK must provide aggregation functionality for each of New Relic's supported metric
   types.  The appropriate aggregation function is different for each metric type.

### Metric Types

The SDK supports three metric types: [Count](#count), [Gauge](#gauge), and [Summary](#summary).

All metric types have these common fields:

| field  | type | notes |
| ------ | ---- | ----- |
| `name` | text | This is the name of a metric. |
| `attributes` | dictionary/map/hash | A map of key/value pairs associated with this metric.  Values can be a string, numeric, or boolean. |
| `timestamp`  | timestamp | A timestamp is required on every metric.  It may be omitted if it is included in the metric [batch](#metric-batch). |

It must be possible to construct a metric with a name and set of attributes.

#### `Count`

  In addition to the common fields, the count metric type has the following field:

  | field | type | notes |
  | ----- | ---- | ----- |
  | `value` | numeric | |
  | `interval` | numeric | Length of the time window.  Must be a positive whole number. |

  It must be possible to construct a count metric with a value.

  When aggregating, it must be possible to `set` the value of an
  existing count metric, and to `increment` the value by an arbitrary amount.

  An example:
  ```python
  def setValue(self, value):
      self.value = value;

  def incrementValue(self, value):
      self.value += value;
  ```

#### `Gauge`

  In addition to the common fields, the gauge metric type has the following fields:

  | field  | type |
  | ------ | ---- |
  | `value` | numeric |

  It must be possible to construct a gauge metric with a value and timestamp.

  When aggregating, it must also be possible to `set` the value of an existing gauge
  metric. When the value is set, the `timestamp` field must be set to the current time.

  An example:
  ```golang
  func (g *Gauge) Value(val float64) {
      g.Value = val
      g.Timestamp = time.Now()
  }
  ```

#### `Summary`

  In addition to the common fields, the summary metric type has the following fields:

  | field  | type |
  | ------ | ---- |
  | `min` | numeric |
  | `max` | numeric |
  | `count` | integer |
  | `sum` | integer |

  It must be possible to construct a summary metric with a min, max, count, and sum.

  When aggregating, it must be possible to adjust all of these value fields together.
  An example:
  ```java
  public void record(double value) {
      this.count += 1;
      this.sum += value;
      this.max = Math.max(this.max, value);
      this.min = Math.min(this.min, value);
  }
  ```
  \*When the fields of a summary are reported to New Relic, they are combined into a
  single JSON object as the `value` field.

### Metric Batch

  A metric batch is a [batch](#batches) that contains only metrics.  Metric batches can
  contain a mixture of metric types.

## Spans



### Span Batch

  A span batch is a [batch](#batches) that contains only spans.

## Batches

A batch is a data structure containing multiple data points of the same telemetry type.
The batch should also contain a map of common attributes that apply to all data points in
the batch.

If the same attribute key exists in both the batch attributes and on a data point within
the batch, the data point's attribute will take precedence in the New Relic back end.

Batches are a convenient way to serialize data into the
[payload](./communication.md#payload) used for communication with New Relic.

### Sending

  The SDK must provide the means to send batches to New Relic in a way that
  handles communication failures using the
  [recommended strategy](./communication.md#graceful-degradation).
