# Communication with New Relic

All communication with New Relic **MUST** take place via the public telemetry ingest APIs. These APIs all share a common JSON format to provide a consistent experience across data types. SDK implementations **MUST** adhere to this common format when sending data to New Relic.

## Request format

The SDK **MUST** use the [Telemetry ingest APIs](https://docs.newrelic.com/docs/data-ingest-apis) to send data to New Relic. The SDK sends all telemetry of a given type to the appropriate telemetry ingest endpoint.

* SDKs **MUST** compress the JSON payload with `gzip` encoding by default.
* Only send API keys as headers (not query params)


### Request ID header

When communicating with data ingest services, there are 3 possible outcomes of the HTTP call:

1. OK status (200 <= status_code < 300), indicating data has been received and persisted.
2. non-OK status, indicating data has not been persisted
3. disconnect (either client or server). In this case, the connection is closed prior to receiving a status indication.

In case (3) above, the client should only retry the request if the request is idempotent since data may or may not have been persisted (and thus the data may get recorded twice, resulting in the data aggregates being inaccurate).

To prevent data loss while allowing clients to retransmit in the case of transient failures, the ingest service must be able to identify duplicate requests; therefore, all SDKs **MUST** send the following HTTP header with the request:

| Header Name | Header Value | Code Example |
| ----------- | ------------ | ------------ |
| x-request-id | A [version 4 UUID string](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) | [str(uuid.uuid4())](https://docs.python.org/3/library/uuid.html#uuid.uuid4) |

**NOTE** the request ID should be generated before the first attempt to send the request is made and the value should be maintained throughout any retries which transmit the same payload. If the SDK partitions the payload in response to a 413 status code, a unique request ID should be used for the transmission of each partition.

### User Agent

All SDKs **MUST** send the following `User-Agent` header by default:
`NewRelic-<language>-TelemetrySDK/<version>`

Additionally, SDKs **MUST** implement an API which allows appending product
name and version to the `User-Agent` header in accordance with
[RFC 7231](https://tools.ietf.org/html/rfc7231#section-5.5.3).

The string appended to the `User-Agent` header **MUST** be in the format: `product/product_version`.

Example:

`NewRelic-Python-TelemetrySDK/0.1 NewRelic-Python-OpenCensus/0.2.1`

```python
def add_version_info(self, product, product_version):
    """Adds product and version information to a User-Agent header

    This method implements
    https://tools.ietf.org/html/rfc7231#section-5.5.3

    :param product: The product name using the SDK
    :type product: str
    :param product_version: The version string of the product in use
    :type product_version: str
    """
    self.user_agent += " {}/{}".format(product, product_version)
```

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
| `413` | Payload too large (`1 MB` limit) | each failure | `split` data and retry | no | See: [splitting data](#splitting-data) |
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

### Splitting data

The New Relic ingest API may return an HTTP 413 (payload too large).  The SDK must ensure
that data that is or would be rejected due to payload size is successfully sent to New Relic.

Some strategies include:

* Preemptively splitting large payloads.
* Splitting and retrying requests in response to an HTTP 413.

If a request results in an HTTP 413, and the payload of that request cannot be split, the
SDK should drop the data. See [dropping data](#dropping-data).
