# Telemetry SDK

The SDK is designed to accept telemetry data send it to New Relic. It's an extension of the public telemetry ingest HTTP API. It's not an agent or tracer, so it doesn’t instrument anything, but it tries to be helpful (and not too clever). It addresses two primary use cases:

1. You want to record telemetry within an application. For example, you record a summary metric with every request and you’d like to just hand the metric you have to the SDK and let it handle the aggregation, batching, and sending.
 
2. You have a "bag of metrics" or other telemetry that something else created (say, Prometheus or OpenCensus) and you want to get them to New Relic efficiently.

## Communication with New Relic

All communication with New Relic **MUST** take place via the public [telemetry ingest APIs](https://source.datanerd.us/ingest/ingest-specs). These APIs all share a common JSON format to provide a consistent experience across data types. SDK implementations **MUST** adhere to this common format when sending data to New Relic.

### Common JSON format

All JSON payloads sent to New Relic **MUST** comply with the following structure:

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

Payloads of different types **MUST NOT** be combined into a single payload. Each payload will be sent to its respective telemetry ingest endpoint.

SDK implementations **SHOULD** use common attributes to reduce the size of repeated attributes in payloads when applicable.

#### Intrinsic attributes

Intrinsic attributes are key/value pairs which make up the definition of the data type. SDK implementations **MUST** apply all intrinsic attributes their respective payloads. The required intrinsic attributes are defined below:

| Data Type | Intrinsic Keys | Spec |
| --- | --- | --- |
| Metric | `metricName`, `metricType`, `value`, `timestamp`, `interval.ms` | [Metric Spec](https://docs.google.com/document/d/1YQniWHGxO6WcOk3AQLOzchiMksNJ3zCY_aPo8Zk8des/edit#heading=h.8y1h1pcnoerc) |
| Span | `traceId`, `guid`, `name`, `category`, `parentId`, `duration.ms`, `timestamp` | [Span Spec]() |
| Log | `message`, `timestamp` | [Log Spec]() |
| Event | `eventType`, `timestamp` | [Event Spec]() |

Further information on each intrinsic and their expected value can be found in the respective spec links above.

#### Custom attributes

SDK implementations **MUST** allow users to supply custom attribute key/value pairs as common attributes as well as attributes on individual data points.

### Request format

When data is sent to New Relic it **MUST** be done as an HTTP POST with additional required headers and the common JSON format body. SDK implementations **SHOULD** compress the JSON payload with `gzip` encoding by default.

**Note:** Some headers **MAY** be sent as a query parameter **instead** of a header where noted below. SDK implementations **SHOULD** use headers whenever possible. If a header and a query parameter with the same name but differing values is detected the request will be rejected.

#### Required headers

| Header | Description | Can be provided as query parameter |
| ------ | ----------- | ---------------------------------- | 
| `Content-Type` | **MUST** be `application/json` | no |
| `Content-Length` | **MUST** be the length of the request body in octets (8-bit bytes) unless sent with chunked encoding. | no |
| `Api-Key` | The value of this header **MUST** be the New Relic Insights Key. | yes |
| `Content-Encoding` | **MUST** be present if the payload compressed. If present the value **MUST** be `gzip`. | no |

#### Optional headers

| Header | Description | Can be provided as query parameter |
| ------ | ----------- | -----------------------------------|
| `Data-Format` | The optional payload data format. **MUST** be present if the format of the data is not the common JSON format, e.g. `zipkin` or `telegraf`. | yes |
| `Data-Format-Version` | The version of the data format. **MUST** be present if the `Data-Format` header is present. | yes |
| `User-Agent` | The client identity. New Relic clients **SHOULD** send this header to report their language and version. | no |

### Response codes

All New Relic telemetry ingest endpoints provide very basic validation for incoming requests with no validation performed on the body of the request. The telemetry ingest endpoints will return a response code corresponding to the validation failure that may have occurred or a `202` response code if it was successful.

#### Telemetry API

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

##### Backoff example

* Harvest Period = `5 seconds`
* Backoff sequence = [`5`, `5`, `10`, `20`, `40`, `80`]

1. Telemetry SDK starts receiving a `500` response code when sending a payload
2. Attempt to send again `5 seconds` later (the harvest frequency) for *2 more tries*
3. Telemetry SDK is still unable to send data, double the wait time and try again
4. Repeat step `3.` until the maximum wait time is hit (`HarvestFrequency * 16`) or data is successfully sent
5. Telemetry SDK continues to receieve `500` errors and will keep attempting to send data every `80 seconds` (in this case) until successful.

For additional information on how to handle data collection and storage when these types of errors occur see [Graceful degradation](#graceful-degradation)


#### Low-level API handling

SDK implementations **SHOULD NOT** perform response code error handling as noted above in the low-level API. However, the low-level API **MUST** propagate this error information to the consumer of that API. This can be done via exceptions, errors or a response object. Additional information can be found in the [Validation](#validation) section below.  


### Graceful degradation

The SDK may be unable to communicate with New Relic for a variety of reasons including network outages, misconfigurations or service outages. The Telemetry API **MUST** provide facilities to gracefully handle these failure cases or allow the consumer to handle them as they see fit.

#### Telemetry API

The telemetry API **MUST** provide the ability to handle continued collection and retention of telemetry data despite a temporary inability to communicate with New Relic.

The SDK **MUST NOT** prevent data collection for any reason. When the SDK is unable to communicate with New Relic and the response code allows for retries (see: [Response codes](#response-codes)) the SDK **MUST** attempt to retain this data using the respective strategy below:

##### 413 - Payload too large #####

When a payload is too large to be accepted by New Relic (i.e. it is above the `1 MB` limit) the SDK **MUST** split this payload in half and retry each half individually. If either payload receives another `413` exception it **MUST** attempt to split once more. If any payload receieves a `413` exception at this stage the data **MUST** be dropped and a notification **SHOULD** be logged.

The payload **SHOULD** be split *after* receiving a 413 response code and should not be done optimistically.

##### 429 - Too many requests #####

New Relic may return this response code if it needs to throttle inbound requests from the SDK. This may happen for a variety of reasons and the SDK **MUST** adhere to the `Retry-After` header that is returned by the telemetry ingest endpoint.

Data collection **MUST** continue to occur, data from the failed batch **MUST** be retained indefinitely and **MAY** be aggregated into the next batch if possible to conserve memory. If the failed batch cannot be aggregated into the next batch it **MUST** be appended to it.

##### All other response codes #####

Any other response code encountered by the SDK may indicate a transient error condition.

Data collection **MUST** continue to occur, data from the failed batch **MUST** be retained indefinitely and **MAY** be aggregated into the next batch if possible to conserve memory. If the failed batch cannot be aggregated into the next batch it **MUST** be appended to it.


#### Low-level API

The low-level API **SHOULD NOT** provide any automatic handling of error cases. However, the low-level API **MUST** propagate information about these failure modes to the consumer of the API. This communication mechanism should follow a pattern that is idiomatic for the respective SDK language and may include exceptions, errors or similar facilities.


### Validation

The New Relic telemetry ingest pipeline performs lightweight synchronous validation on all inbound requests as well as a more thorough asynchronous validation on the payload contents. SDK implementations **SHOULD** only perform minimal validation on input data as noted below.

#### SDK validation

In an effort to keep SDK complexity low the SDK implementation **MUST** limit its validation to the bare minimum required to be safe for clients. The SDK **MUST NOT** throw exceptions or errors on data ingest unless the caller is explicitly notified of the possibility. 

Data **MUST** always be accepted and sent to the telemetry ingest APIs unless storing, marshalling or sending it would result in an immediate exception. In cases where the data is invalid it **MUST** be dropped.

For example, in languages where `NaN` or `Infinity` can be represented these values may be stored but can not be correctly marshalled to JSON and thus are dropped when JSON marshalling occurs because it violates the safety of the payload.

#### New Relic backend validation

The New Relic telemetry ingest pipline has its own set of limits and restrictions on inbound data that it enforces at various stages of data ingest.

Initial lightweight validation of inbound requests occur synchronously and result in [Response codes](#response-codes) being sent back to the SDK to indicate failed validation.

A more thorough validation of payload contents occurs asynchronously the SDK will not be directly notified of this failure. Instead, a custom event named `NrIntegrationError` is emitted to the account that includes data about the failure. Some potentital failure cases are listed below:

| Failure | Discard data point | Discard payload |
| ------- | -------------------| ----------------|
| Invalid JSON | n/a | yes |
| Invalid `common` section | n/a | yes |
| Invalid or missing required JSON keys on data point (e.g. - `metricType`, `timestamp`) | yes | no |
| Payload timestamp is not within the last 48 hours (in either direction) | yes | n/a |

##### New Relic storage constraints

The New Relic backend has limits on the size and number of attributes that may be stored for data points. Detailed explanations of these limits can be found here: [Metric API Limits](https://docs.google.com/document/d/1YQniWHGxO6WcOk3AQLOzchiMksNJ3zCY_aPo8Zk8des/edit#heading=h.bp5nzjycedpw)


![image](high-level-MELT-architecture.png)

## Configuration

SDK implementations **MUST** allow for configuration of the following options:

1. `API key`
	* The API is required in order to communicate with the New Relic telemetry ingest APIs. 
2. `Host URL override`
	* To facilitate communication with alternative New Relic backends as well as allowing for simple integration testing with a mock backend the SDK should allow each ingest URL to be overridden.
3. `Harvest interval`
	* The harvest interval should default to `5 seconds` with the ability for consumers of the SDK to set a custom interval in `seconds`.

## Safety

### Buffering limits

When data cannot be sent to New Relic, SDK implementations **SHOULD** limit the amount of data that is buffered before discarding to prevent excessive memory usage.

#### Memory limit

If the SDK language can support simple, memory constrained buffers for data points it **SHOULD** use them and **MUST** limit the buffer size to `2 MB` with no limit on the number of items.

#### Data point limit

If the SDK language is unable to easily support memory constrained buffers it **SHOULD** use an item limited buffer and **MUST** limit the buffer size to `2000` items.

### No-op behavior

When the Telemetry API is in use by customers it will lead to API calls being used (possibly extensively) throughout customer code. If the customer needs to quickly disable these API calls the SDK **MUST** provide a no-op implementation that the customer can swap in without requiring that they modify all API call sites.

By providing a no-op implementation this means that any call to the Telemetry API with effectively have zero cost until a real implementation in swapped back in. 


## Logging

SDK implementations **MUST** write troubleshooting and error information to a log file by whatever means is the most idiomatic for the language. SDKs **MAY** choose to write to an existing application log or they **MAY** log to their own file.

Logs should be used judiciously and generally only in exceptional error cases or when needed for SDK supportability.


# Telemetry SDK API

The Telemetry SDK suits these needs with two levels of API.

![image](telemetry-sdk-API-levels.png)

## Telemetry API

 The top-level telemetry API offers a harvester that can aggregate, batch, and send the data to New Relic on a regular interval. It handles communication with New Relic and will automatically retry requests, split overly large payloads, and backoff in the face of rate limiting. This level showcases the best practices for sending data to New Relic and is what almost everyone should use.

### Harvester
* harvester is shared across telemetry types
* methods to record metrics and spans
* decouple aggregation and batching
* error handling - this level is more likely to be used from application code, make it impossible for the user to handle errors

#### Batching
* The harvester batches telemetry using the configured interval

#### Aggregation
* The harvester handles aggregation for metrics
* Metrics are aggregated differently by type, see below

## Low-level API

While the telemetry API tries to be helpful, it is supported by a low-level “no frills” API that won’t do anything unless asked explicitly. It knows about the New Relic data structures, like spans and summary metrics. It can accept a batch of telemetry data and return an HTTP request object or send it for you and return the response object. If you really need to directly manipulate and send data without built-in aggregation, this API offers that full control.

### Batches
* A batch has methods to accept telemetry
* It can return a request object or make the call itself and return the response
* No error handling (say why)

### Metrics
* To what extent is this just documenting the metrics backend?
* Do we want to talk about attributes here? Limits/cardinality/how common attributes work?

##### Metric Identities
* Metrics are uniquely defined and aggregated by their `name`, `type` and `attribute keys + values`

#### Gauge

#### Delta count

#### Cumulative count
* Special logic here

#### Summary

### Spans
* To what extent is this just documenting the traces backend?


