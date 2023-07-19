<a href="https://opensource.newrelic.com/oss-category/#community-project"><picture><source media="(prefers-color-scheme: dark)" srcset="https://github.com/newrelic/opensource-website/raw/main/src/images/categories/dark/Community_Project.png"><source media="(prefers-color-scheme: light)" srcset="https://github.com/newrelic/opensource-website/raw/main/src/images/categories/Community_Project.png"><img alt="New Relic Open Source community project banner." src="https://github.com/newrelic/opensource-website/raw/main/src/images/categories/Community_Project.png"></picture></a>

# Telemetry SDK specs
The telemetry SDK specs is a documentation repository that describes the basic features
shared across language implementations of the New Relic Telemetry SDKs. You should use
these specs to help you understand what is commonly included in the Telemetry SDKs, why
they were designed the way they are, and how they work so that you can extend them and contribute
effectively.

## The Telemetry SDK
The New Relic Telemetry SDKs are designed to accept telemetry data and send it to
New Relic. They are a semantic extension of the New Relic public-facing HTTP Telemetry
Ingest API, which is a common interface for multiple telemetry backends. They are _not_
agents or tracers, and they don't instrument anything. They try to be helpful, so the
user's job of sending telemetry data to New Relic can be done in the right way, easily.
Lastly, they are extensible and open source, so they can be tailored for each use case.


The Telemetry SDK's primary use case is:

A user has telemetry data: metrics, traces, logs, or events, that have already been created
(for example: from Prometheus or OpenTelemetry), and they want to get them into New Relic
easily and efficiently.  The Telemetry SDK can be used to quickly write an integration
with the New Relic Telemetry Ingest API.

As New Relic's Telemetry Ingest API is public facing, with user documentation, a
customer could interact with it using a simple HTTP Client in any language.  The values
that the telemetry SDKs provide over an HTTP Client are:

* They provide language-appropriate abstractions for New Relic's various metric types and
  traces.
* They handle communication with New Relic:
  * They know the correct URLs for metrics, traces, etc...
  * They know how to provide authentication information in HTTP requests.
  * They implement New Relic's recommended handling of failed requests.

## Contents

* [Capabilities](./capabilities.md)
* [Communication with New Relic](./communication.md)
* [Validation](./validation.md)
* [Configuration & Logging](./configuration-and-logging.md)
* [Limits](./limits.md)

## Available Telemetry SDK Repositories
* [Java](https://github.com/newrelic/newrelic-telemetry-sdk-java)
* [Python](https://github.com/newrelic/newrelic-telemetry-sdk-python)
* [Go](https://github.com/newrelic/newrelic-telemetry-sdk-go)
* [.NET](https://github.com/newrelic/newrelic-telemetry-sdk-dotnet)
* [Node](https://github.com/newrelic/newrelic-telemetry-sdk-node)
* [C](https://github.com/newrelic/newrelic-telemetry-sdk-c)
* [Ruby](https://github.com/newrelic/newrelic-telemetry-sdk-ruby)
* [Rust (alpha)](https://github.com/newrelic/newrelic-telemetry-sdk-rust)

## Telemetry data conversion
How New Relic ingests and stores telemetry data can be different from other telemetry
systems. How we convert non New Relic telemetry data into a New Relic-friendly format is
documented in the [New Relic Exporter Specifications](https://github.com/newrelic/newrelic-exporter-specs).
