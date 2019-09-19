# Communication with New Relic

All communication with New Relic **MUST** take place via the public telemetry ingest APIs. These APIs all share a common JSON format to provide a consistent experience across data types. SDK implementations **MUST** adhere to this common format when sending data to New Relic.

## Request format

The SDK **MUST** use the [Telemetry ingest APIs](https://docs.newrelic.com/docs/data-ingest-apis) to send data to New Relic. The SDK sends all telemetry of a given type to the appropriate telemetry ingest endpoint.

* SDKs **MUST** compress the JSON payload with `gzip` encoding by default.
* Only send API keys as headers (not query params)
* User-Agent string: `NewRelic-<language>-TelemetrySDK/<version>`

### Payload

Payloads of different telemetry types cannot be combined.

All JSON payloads sent to New Relic **MUST** use the New Relic common format.
This is an example of the common format:

```
[
  {
    "common": {
      <intrinsic attributes>
      "attributes" : {
          <custom attributes>
        }
    },
    "<spans|logs|metrics|events>" : [
      {
        <intrinsic attributes>,
        "timestamp": 1522434601409,
        "attributes" : {
          <custom attributes>
        }
      },
      {
        <intrinsic attributes>,
        "timestamp": 1522434601409,
        "attributes" : {
          <custom attributes>
        }
      } ]
  }
]
```

SDK implementations **SHOULD** use the top-level common block to reduce the size of repeated attributes in payloads when applicable.

## Response codes

The telemetry ingest API validates the basic shape of the request without looking at the POST body. [Its responses are documented here](https://docs.newrelic.com/docs/data-ingest-apis/get-data-new-relic/metric-api/report-metrics-metric-api#response-status-codes).

SDK implementations must perform response code error handling in the Telemetry API as documented below. The telemetry API should provide a mechanism for the consumer of this API to be notified (or react to) any error conditions that may occur rather than hiding all errors from the user.

| Response code | Description | Log error | Retry behavior | Drop data | Other |
| ------------- | ------------| --------- | ---------------| ----------|------|
| `200 - 299` | Successful request | | | | |
| `400` | Generally invalid request | once | no | yes | See: [dropping data](#dropping-data). |
| `401` | Unauthorized | once | no | yes | See: [dropping data](#dropping-data). |
| `403` | Authentication failure | once | no | yes | See: [dropping data](#dropping-data). |
| `404` | Incorrect path | once | no | yes | See: [dropping data](#dropping-data). |
| `405` | Incorrect HTTP method (`POST` required) | once | no | yes | See: [dropping data](#dropping-data). Should never occur in the Telemetry SDK but should still be handled |
| `408` | Request timeout | each failure | yes | not yet | |
| `409` | Conflict | once | no | yes | See: [dropping data](#dropping-data). |
| `410` | Gone | once | no | yes | See: [dropping data](#dropping-data). |
| `411` | Missing `Content-Length` header | once | no | yes | See: [dropping data](#dropping-data). Should never occur in the Telemetry SDK but should still be handled |
| `413` | Payload too large (`1 MB` limit) | each failure | `split` data and retry | no |
| `429` | Too many requests | each failure | Retry based on `Retry-After` response header | no | `Retry-After` (`integer`) for how long wait until next retry in `seconds` |
| `Anything else` | Unknown | each failure | Retry with backoff | not yet | See [graceful degradation](#graceful-degradation). |

### Graceful degradation

The SDK may be unable to communicate with New Relic for a variety of reasons including
network outages, misconfiguration or service outages. Telemetry SDKs must provide
facilities to gracefully handle these failure cases or allow the consumer to handle them
as they see fit.  The SDKs must also provide functionality to make a request with no
response handling or retrying.

The recommended handling of failed requests to the ingest API is to retry the request at
increasing intervals and to eventually drop data if the request cannot be completed.

The amount of time to wait after a request can be computed using this logic:
```
MIN(backoff_max, backoff_factor * (2 ^ (number_of_retries - 1)))
```

For a _backoff factor_ of 1 second, and a _backoff max_ of 16 seconds, the retry delay
interval should follow a pattern of [0, 1, 2, 4, 8, 16, 16, ...]. Subsequent retries should
wait 16 seconds until the request has been retried the configured _max retries_ number of times.

The _total retry duration_ can be computed from the combination of _backoff factor_ and
_backoff max_.  SDKs may provide a function to configure retry behavior by specifying the
_total retry duration_ instead of _max retries_.

#### Backoff example:

* Backoff factor = `5 seconds`
* Backoff max = `80 seconds`
* Max retries = `8`
* Backoff sequence = [`0`, `5`, `10`, `20`, `40`, `80`, `80`, `80`]

1. The telemetry SDK attempts to send a payload at t=13:00:00, and receives a `500` response.
1. The telemetry SDK attempts to send again at
    * +0 : 13:00:00
    * +5 : 13:00:05
    * +10 : 13:00:15
    * +20 : 13:00:35
    * +40 : 13:01:15
    * +80 : 13:02:35
    * +80 : 13:03:55
    * +80 : 13:05:15
    * -- max retries exceeded. The data in this request should be dropped. See [dropping data](#dropping-data).

### Dropping data

Whenever dropping data, the SDK must emit an error level log statement indicating the
number of data points dropped.

SDKs should not attempt to merge a failed payload with the rest of the data being stored
by the SDK.

SDKs may provide functionality for users to provide their own handler for dropped data, so
that a user of the SDK may merge unsent data back into their own data collector in the
way that makes sense for their use case.

### Telemetry API

The telemetry API must provide the ability to handle continued collection and retention of telemetry data despite a temporary inability to communicate with New Relic.

The SDK must not prevent data collection for any reason. When the SDK is unable to communicate with New Relic and the response code allows for retries (see: [Response codes](#response-codes)) the SDK must attempt to retain this data using the respective strategy below:

#### 413 - Payload too large

When a payload is too large to be accepted by New Relic (i.e. it is above the `1 MB` limit)
the SDK must split this payload into multiple batches and retry each batch individually.
If any of these batches receives another `413` exception, the SDK must attempt to split it
once more. If any payload receieves a `413` exception at this stage the data should be
dropped.  See [dropping data](#dropping-data).

The payload should be split *after* receiving a 413 response code and should not be done optimistically.

Retry state for each half should be maintained individually. For example, a `500` returned
for the first half will not increment the failure count for the second half.


Retry state is scoped on a per-request basis, and persists in the case of nonsequential
response codes. For example, if the SDK encounters three `500`s, then a `413`, the split
payload requests will have a 4x multiplier for the next `500` in the chain.

#### 429 - Too many requests

New Relic may return this response code if it needs to throttle inbound requests from the
SDK. This may happen for a variety of reasons and the SDK must adhere to the `Retry-After`
header that is returned by the telemetry ingest endpoint.

The SDK must respect the value of the `Retry-After` header.  If the value of the
`Retry-After` header is greater than the total backoff time, the data must be dropped.
The SDK must not attempt to send the same data again before `Retry-After` time has elapsed.

#### All other response codes

Any other response code encountered by the SDK may indicate a transient error condition.

#### Any other connection failure

For any other connection failure, the SDK should follow [the backoff strategy above](#graceful-degradation).
