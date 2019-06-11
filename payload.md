### Common JSON format

The SDK sends all telemetry of a given type to the appropriate telemetry ingest endpoint. Payloads of different types cannot be combined.

All JSON payloads sent to New Relic **MUST** use the [New Relic common format](https://source.datanerd.us/ingest/ingest-specs/blob/master/nrCommonFormat.md).

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
