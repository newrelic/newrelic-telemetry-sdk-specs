# Communication with New Relic

All communication with New Relic **MUST** take place via the public [telemetry ingest APIs](https://source.datanerd.us/ingest/ingest-specs). These APIs all share a common JSON format to provide a consistent experience across data types. SDK implementations **MUST** adhere to this common format when sending data to New Relic.


## Request format

The SDK **MUST** use the [Telemetry ingest API](https://source.datanerd.us/ingest/ingest-specs/blob/master/ingestConsistency.md) to send data to New Relic. 

* SDKs **MUST** compress the JSON payload with `gzip` encoding by default. 
* Only send API keys as headers (not query params)
* User-Agent string: `NewRelic-<language>-TelemetrySDK/<version>`

## Response codes

Vortex validates the basic shape of the request without looking at the POST body. [Its responses are documented here](https://source.datanerd.us/ingest/runbooks/blob/master/vortex/vortex-responses.md).

SDK implementations **MUST** perform response code error handling in the Telemetry API as documented below. The telemetry API **SHOULD** provide a mechanism for the consumer of this API to be notified (or react to) any error conditions that may occur rather than hiding all errors from the user.

| Response code | Description | Log error | Retry behavior | Drop data | Other |
| ------------- | ------------| --------- | ---------------| ----------|------|
| `202` | Successful request | | | | |
| `400` | Generally invalid request | once | no | yes | |
| `403` | Authentication failure | once | no | yes | |
| `404` | Incorrect path | once | no | yes | |
| `405` | Incorrect HTTP method (`POST` required) | once | no | yes | Should never occur in the Telemetry API or Low-level API but should still be handled |
| `411` | Missing `Content-Length` header | once | no | yes | Should never occur in the Telemetry API or Low-level API but should still be handled |
| `413` | Payload too large (`1 MB` limit) | each failure | `split` data and retry | no |
| `429` | Too many requests | each failure | Retry based on `Retry-After` response header | no | `Retry-After` (`integer`) for how long wait until next retry in `seconds` |
| `Anything else` | Unknown | each failure | Retry with backoff | no | Backoff sequence (`H` = Harvest period in `seconds`)<br><br>[`H*1`, `H*1`, `H*2`, `H*4`, `H*8`, `H*16` (repeat `H*16` until success)] |

#### Backoff example

* Harvest Period = `5 seconds`
* Backoff sequence = [`5`, `5`, `10`, `20`, `40`, `80`]

1. Telemetry SDK starts receiving a `500` response code when sending a payload
2. Attempt to send again `5 seconds` later (the harvest frequency) for *2 more tries*
3. Telemetry SDK is still unable to send data, double the wait time and try again
4. Repeat step `3.` until the maximum wait time is hit (`HarvestFrequency * 16`) or data is successfully sent
5. Telemetry SDK continues to receieve `500` errors and will keep attempting to send data every `80 seconds` (in this case) until successful.

For additional information on how to handle data collection and storage when these types of errors occur see [Graceful degradation](#graceful-degradation)

### Low-level API handling

SDK implementations **SHOULD NOT** perform response code error handling as noted above in the low-level API. However, the low-level API **MUST** propagate this error information to the consumer of that API. This can be done via exceptions, errors or a response object. Additional information can be found in the [Validation](#validation) section below.  



## Graceful degradation

The SDK may be unable to communicate with New Relic for a variety of reasons including network outages, misconfigurations or service outages. The Telemetry API **MUST** provide facilities to gracefully handle these failure cases or allow the consumer to handle them as they see fit.

### Telemetry API

The telemetry API **MUST** provide the ability to handle continued collection and retention of telemetry data despite a temporary inability to communicate with New Relic.

The SDK **MUST NOT** prevent data collection for any reason. When the SDK is unable to communicate with New Relic and the response code allows for retries (see: [Response codes](#response-codes)) the SDK **MUST** attempt to retain this data using the respective strategy below:

#### 413 - Payload too large #####

When a payload is too large to be accepted by New Relic (i.e. it is above the `1 MB` limit) the SDK **MUST** split this payload in half and retry each half individually. If either payload receives another `413` exception it **MUST** attempt to split once more. If any payload receieves a `413` exception at this stage the data **MUST** be dropped and a notification **SHOULD** be logged.

The payload **SHOULD** be split *after* receiving a 413 response code and should not be done optimistically.

#### 429 - Too many requests #####

New Relic may return this response code if it needs to throttle inbound requests from the SDK. This may happen for a variety of reasons and the SDK **MUST** adhere to the `Retry-After` header that is returned by the telemetry ingest endpoint.

Data collection **MUST** continue to occur, data from the failed batch **MUST** be retained indefinitely and **MAY** be aggregated into the next batch if possible to conserve memory. If the failed batch cannot be aggregated into the next batch it **MUST** be appended to it.

#### All other response codes #####

Any other response code encountered by the SDK may indicate a transient error condition.

Data collection **MUST** continue to occur, data from the failed batch **MUST** be retained indefinitely and **MAY** be aggregated into the next batch if possible to conserve memory. If the failed batch cannot be aggregated into the next batch it **MUST** be appended to it.

### Low-level API

The low-level API **SHOULD NOT** provide any automatic handling of error cases. However, the low-level API **MUST** propagate information about these failure modes to the consumer of the API. This communication mechanism should follow a pattern that is idiomatic for the respective SDK language and may include exceptions, errors or similar facilities.
