# Limits

## Buffering limits

When data cannot be sent to New Relic, SDK implementations **SHOULD** limit the amount of data that is buffered before discarding to prevent excessive memory usage.

### Memory limit

If the SDK language can support simple, memory constrained buffers for data points it **SHOULD** use them and **MUST** limit the buffer size to `2 MB` with no limit on the number of items.

### Data point limit

If the SDK language is unable to easily support memory constrained buffers it **SHOULD** use an item limited buffer and **MUST** limit the buffer size to `2000` items.
