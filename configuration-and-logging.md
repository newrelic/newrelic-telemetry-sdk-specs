# Configuration

SDK implementations must allow for configuration of the following options:

1. `API key`
  * The API is required in order to communicate with the New Relic telemetry ingest APIs.
2. `Host URL override`
  * To facilitate communication with alternative New Relic backends as well as allowing for simple integration testing with a mock backend the SDK should allow each ingest URL to be overridden.
3. `Harvest interval`
  * The harvest interval should default to `5 seconds` with the ability for consumers of the SDK to set a custom interval in `seconds`.
4. `Failed request retrying`
  * It must be possible to configure the _backoff factor_, that is the amount of time to wait
    after a failed request before attempting to send the request again.
  * It must also be possible to configure the _max retries_.
    See [communication backoff](./communication.md#graceful-degradation).
5. `Logging`
  * Whether the SDK logs at all, the verbosity, and the destination of the log must all be configurable.
5. `Audit logging enabled`
  * If audit logging is enabled, the SDK should record additional highly verbose debugging information at the `DEBUG` logging level.  The default value for this setting must be `false`.

# Logging

SDK implementations should log troubleshooting and error information by whatever means is
the most idiomatic for the language. SDKs must provide a mechanism to configure the
destination and verbosity of the log, and a way to disable the SDK's logging.

When enabled, SDKs should primarily use three logging levels: `ERROR`,`INFO`, `DEBUG`,
with `DEBUG` being the most verbose and `ERROR` the least.  SDKs should log:

* `ERROR` level messages only in error cases
  * _example_: if the connection to New Relic is refused
  ```
  ERROR : Failed to open TCP connection to #{config.host_url} (Connection refused - connect(2))
  ```
* `INFO` level messages sparingly
  * _example_: on startup after the config has been processed listing the configured options:
  ```
  INFO : New Relic Telemetry SDK started
  INFO : Connecting to #{config.host_url}
  INFO : Harvesting every #{config.harvest_interval} seconds
  ```
* `DEBUG` level messages as much as necessary for SDK supportability
  * _example_: when handling responses from the Metric API backend:
  ```
  DEBUG : Reported 403, Forbidden
  DEBUG : Reported 202, Accepted
  ```

If audit logging is enabled in the SDK configuration, additional highly verbose debugging information should be logged at the `DEBUG` level:
  * _example_: every payload sent to the Metric API backend:
  ```
  DEBUG : Sent payload: '{ ... a large json payload here ...}'
  ```
