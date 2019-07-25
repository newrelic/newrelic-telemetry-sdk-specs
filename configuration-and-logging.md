# Configuration

SDK implementations **MUST** allow for configuration of the following options:

1. `API key`
  * The API is required in order to communicate with the New Relic telemetry ingest APIs. 
1. `Host URL override`
  * To facilitate communication with alternative New Relic backends as well as allowing for simple integration testing with a mock backend the SDK should allow each ingest URL to be overridden.
1. `Harvest interval`
  * The harvest interval should default to `5 seconds` with the ability for consumers of the SDK to set a custom interval in `seconds`.
1. `Request timeout`
  * The total amount of time to keep retrying a failed request.  

## No-op behavior

When the Telemetry API is in use by customers it will lead to API calls being used (possibly extensively) throughout customer code. If the customer needs to quickly disable these API calls the SDK **MUST** provide a no-op implementation that the customer can swap in without requiring that they modify all API call sites.

By providing a no-op implementation this means that any call to the Telemetry API with effectively have zero cost until a real implementation is swapped back in. 

# Logging

SDK implementations **MUST** write troubleshooting and error information to a log file by whatever means is the most idiomatic for the language. SDKs **MAY** choose to write to an existing application log or they **MAY** log to their own file.

Logs should be used judiciously and generally only in exceptional error cases or when needed for SDK supportability.
