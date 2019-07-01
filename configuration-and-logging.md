# Configuration

SDK implementations must allow for configuration of the following options:

1. `API key`
  * The API is required in order to communicate with the New Relic telemetry ingest APIs. 
2. `Host URL override`
  * To facilitate communication with alternative New Relic backends as well as allowing for simple integration testing with a mock backend the SDK should allow each ingest URL to be overridden.
3. `Harvest interval`
  * The harvest interval should default to `5 seconds` with the ability for consumers of the SDK to set a custom interval in `seconds`.
4. `Logging`
  * Whether the SDK logs at all, the verbosity, and the destination of the log should all be configurable.

## No-op behavior

When the Telemetry API is in use by customers it will lead to API calls being used (possibly extensively) throughout customer code. If the customer needs to quickly disable these API calls the SDK must provide a no-op implementation that the customer can swap in without requiring that they modify all API call sites.

By providing a no-op implementation this means that any call to the Telemetry API with effectively have zero cost until a real implementation is swapped back in. 

# Logging

SDK implementations should log troubleshooting and error information by whatever means is the most idiomatic for the language. SDKs must provide a mechanism to configure the destination and verbosity of the log, and a way to disable the SDK's logging. 
When using the default configuration, the SDK should not log.

When enabled, SDKs should primarily use three logging levels: `ERROR`,`INFO`, `DEBUG`, 
with `DEBUG` being the most verbose and `ERROR` the least.  SDKs should log:

* `ERROR` level messages only in exceptional error cases
* `INFO` level messages sparingly
* `DEBUG` level messages as much as necessary for SDK supportability
