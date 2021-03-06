# Change log


All notable changes to the LaunchDarkly Java SDK will be documented in this file. This project adheres to [Semantic Versioning](http://semver.org).

## [4.10.1] - 2020-01-06
### Fixed:
- The `pom.xml` dependencies were incorrectly specifying `runtime` scope rather than `compile`, causing problems for application that did not have their own dependencies on Gson and SLF4J. ([#151](https://github.com/launchdarkly/java-client/issues/151))

## [4.10.0] - 2019-12-13
### Added:
- Method overloads in `ArrayBuilder`/`ObjectBuilder` to allow easily adding values as booleans, strings, etc. rather than converting them to `LDValue` first.

### Changed:
- The SDK now generates fewer ephemeral objects on the heap from flag evaluations, by reusing `EvaluationReason` instances that have the same properties.

### Fixed:
- In rare circumstances (depending on the exact data in the flag configuration, the flag's salt value, and the user properties), a percentage rollout could fail and return a default value, logging the error "Data inconsistency in feature flag ... variation/rollout object with no variation or rollout". This would happen if the user's hashed value fell exactly at the end of the last "bucket" (the last variation defined in the rollout). This has been fixed so that the user will get the last variation.

### Deprecated:
- Deprecated `LDCountryCode`, `LDUser.Builder.country(LDCountryCode)`, and `LDUser.Builder.privateCountry(LDCountryCode)`. `LDCountryCode` will be removed in the next major release, for setting the `country` user property, applications should use `LDUser.Builder.country(String)` and `LDUser.Builder.privateCountry(String)` instead.
- `SegmentRule` is an internal implementation class that was accidentally made public.
- `NullUpdateProcessor` should not be referenced directly and will be non-public in the future; use the factory methods in `Components` instead.


## [4.9.1] - 2019-11-20
### Changed:
- Improved memory usage and performance when processing analytics events: the SDK now encodes event data to JSON directly, instead of creating intermediate objects and serializing them via reflection.

### Fixed:
- A bug introduced in version 4.9.0 was causing event delivery to fail if a user was created with the `User(string)` constructor, instead of the builder pattern.


## [4.9.0] - 2019-10-18
This release adds the `LDValue` class (in `com.launchdarkly.client.value`), which is a new abstraction for all of the data types supported by the LaunchDarkly platform. Since those are the same as the JSON data types, the SDK previously used the Gson classes `JsonElement`, `JsonObject`, etc. to represent them. This caused two problems: the public APIs are dependent on Gson, and the Gson object and array types are mutable so it was possible to accidentally modify values that are being used elsewhere in the SDK.

While the SDK still uses Gson internally, all references to Gson types in the API are now deprecated in favor of equivalent APIs that use `LDValue`. Developers are encouraged to migrate toward these as soon as possible; the Gson classes will be removed from the API in a future major version. If you are only using primitive types (boolean, string, etc.) for your feature flags and user attributes, then no changes are required.

There are no other changes in this release.

### Added:
- `LDValue` (see above).
- The new `jsonValueVariation` and `jsonValueVariationDetail` methods in `LDClient`/`LDClientInterface` are equivalent to `JsonVariation` and `JsonVariationDetail`, but use `LDValue`.

### Deprecated:
- In `LDClient`/`LDClientInterface`: `jsonVariation`/`jsonVariationDetail`. Use `jsonValueVariation`/`jsonValueVariationDetail`.
- In `LDClient`/`LDClientInterface`: `track(String, LDUser, JsonElement)` and `track(String, LDUser, JsonElement, double)`. Use `trackData(String, LDUser, LDValue)` and `trackMetric(String, LDUser, LDValue, double)`. The names are different to avoid compile-time ambiguity since both `JsonElement` and `LDValue` are nullable types.
- In `LDUserBuilder`: `custom(String, JsonElement)` and `privateCustom(String, JsonElement)`. Use the `LDValue` overloads.
- In `LDValue`: `fromJsonElement`, `unsafeFromJsonElement`, `asJsonElement`, `asUnsafeJsonElement`. These are provided for compatibility with code that still uses `JsonElement`, but will be removed in a future major version.


## [4.8.1] - 2019-10-17
### Fixed:
- The NewRelic integration was broken when using the default uberjar distribution, because the SDK was calling `Class.forName()` for a class name that was accidentally transformed by the Shadow plugin for Gradle. ([#171](https://github.com/launchdarkly/java-server-sdk/issues/171))
- Streaming connections were not using the proxy settings specified by `LDConfig.Builder.proxy()` and `LDConfig.Builder.proxyAuthenticator()`. ([#172](https://github.com/launchdarkly/java-server-sdk/issues/172))
- The SDK was creating an unused `OkHttpClient` instance as part of the static `LDConfig` instance used by the `LDClient(String)` constructor. This has been removed.
- Passing a null `sdkKey` or `config` to the `LDClient` constructors would always throw a `NullPointerException`, but it did not have a descriptive message. These exceptions now explain which parameter was null.

## [4.8.0] - 2019-09-30
### Added:
- Added support for upcoming LaunchDarkly experimentation features. See `LDClient.track(String, LDUser, JsonElement, double)`.

### Changed:
- Updated documentation comment for `intVariation` to clarify the existing rounding behavior for floating-point values: they are rounded toward zero.

## [4.7.1] - 2019-08-19
### Fixed:
- Fixed a race condition that could cause a `NumberFormatException` to be logged when delivering event data to LaunchDarkly (although the exception did not prevent the events from being delivered).

## [4.7.0] - 2019-08-02
### Added:
- In `RedisFeatureStoreBuilder`, the new methods `database`, `password`, and `tls` allow you to specify the database number, an optional password, and whether to make a secure connection to Redis. This is an alternative to specifying them as part of the Redis URI, e.g. `rediss://:PASSWORD@host:port/NUMBER`, which is also supported (previously, the database and password were supported in the URI, but the secure `rediss:` scheme was not).
- `LDConfig.Builder.sslSocketFactory` allows you to specify a custom socket factory and truststore for all HTTPS connections made by the SDK. This is for unusual cases where your Java environment does not have the proper root CA certificates to validate LaunchDarkly's certificate, or you are connecting through a secure proxy that has a self-signed certificate, and you do not want to modify Java's global truststore.

### Deprecated:
- `LDConfig.Builder.samplingInterval` is now deprecated. The intended use case for the `samplingInterval` feature was to reduce analytics event network usage in high-traffic applications. This feature is being deprecated in favor of summary counters, which are meant to track all events.

## [4.6.6] - 2019-07-10
### Fixed:
- Under conditions where analytics events are being generated at an extremely high rate (for instance, if an application is evaluating a flag repeatedly in a tight loop on many threads), a thread could be blocked indefinitely within the `Variation` methods while waiting for the internal event processing logic to catch up with the backlog. The logic has been changed to drop events if necessary so threads will not be blocked (similar to how the SDK already drops events if the size of the event buffer is exceeded). If that happens, this warning message will be logged once: "Events are being produced faster than they can be processed; some events will be dropped". Under normal conditions this should never happen; this change is meant to avoid a concurrency bottleneck in applications that are already so busy that thread starvation is likely.

## [4.6.5] - 2019-05-21
### Fixed
- The `LDConfig.Builder` method `userKeysFlushInterval` was mistakenly setting the value of `flushInterval` instead. (Thanks, [kutsal](https://github.com/launchdarkly/java-server-sdk/pull/163)!)

### Added
- CI tests now run against Java 8, 9, 10, and 11.

## [4.6.4] - 2019-05-01
### Changed
- Changed the artifact name from `com.launchdarkly:launchdarkly-client` to `com.launchdarkly:launchdarkly-java-server-sdk`
- Changed repository references to use the new URL

There are no other changes in this release. Substituting `launchdarkly-client` version 4.6.3 with `launchdarkly-java-server-sdk` version 4.6.4 will not affect functionality.

## [4.6.3] - 2019-03-21
### Fixed
- The SDK uberjars contained some JSR305 annotation classes such as `javax.annotation.Nullable`. These have been removed. They were not being used in the public API anyway. ([#156](https://github.com/launchdarkly/java-server-sdk/issues/156))
- If `track` or `identify` is called without a user, the SDK now logs a warning, and does not send an analytics event to LaunchDarkly (since it would not be processed without a user).
### Note on future releases

The LaunchDarkly SDK repositories are being renamed for consistency. This repository is now `java-server-sdk` rather than `java-client`.

The artifact names will also change. In the 4.6.3 release, the generated artifact was named `com.launchdarkly.client:launchdarkly-client`; in all future releases, it will be `com.launchdarkly.client:launchdarkly-java-server-sdk`.

## [4.6.2] - 2019-02-21
### Fixed
- If an unrecoverable `java.lang.Error` is thrown within the analytics event dispatching thread, the SDK will now log the error stacktrace to the configured logger and then disable event sending, so that all further events are simply discarded. Previously, the SDK could be left in a state where application threads would continue trying to push events onto a queue that was no longer being consumed, which could block those threads. The SDK will not attempt to restart the event thread after such a failure, because an `Error` typically indicates a serious problem with the application environment.
- Summary event counters now use 64-bit integers instead of 32-bit, so they will not overflow if there is an extremely large volume of events.
- The SDK's CI test suite now includes running the tests in Windows.

## [4.6.1] - 2019-01-14
### Fixed
- Fixed a potential race condition that could happen when using a DynamoDB or Consul feature store. The Redis feature store was not affected.


## [4.6.0] - 2018-12-12
### Added:
- The SDK jars now contain OSGi manifests which should make it possible to use them as bundles. The default jar requires Gson and SLF4J to be provided by other bundles, while the jar with the "all" classifier contains versions of Gson and SLF4J which it both exports and imports (i.e. it self-wires them, so it will use a higher version if you provide one). The "thin" jar is not recommended in an OSGi environment because it requires many dependencies which may not be available as bundles.
- There are now helper classes that make it much simpler to write a custom `FeatureStore` implementation. See the `com.launchdarkly.client.utils` package. The Redis feature store has been revised to use this code, although its functionality is unchanged except for the fix mentioned below.
- `FeatureStore` caching parameters (for Redis or other databases) are now encapsulated in the `FeatureStoreCacheConfig` class.

### Changed:
- The exponential backoff behavior when a stream connection fails has changed as follows. Previously, the backoff delay would increase for each attempt if the connection could not be made at all, or if a read timeout happened; but if a connection was made and then an error (other than a timeout) occurred, the delay would be reset to the minimum value. Now, the delay is only reset if a stream connection is made and remains open for at least a minute.

### Fixed:
- The Redis feature store would incorrectly report that it had not been initialized, if there happened to be no feature flags in your environment at the time that it was initialized.

### Deprecated:
- The `RedisFeatureStoreBuilder` methods `cacheTime`, `refreshStaleValues`, and `asyncRefresh` are deprecated in favor of the new `caching` method which sets these all at once.

## [4.5.1] - 2018-11-21
### Fixed:
- Fixed a build error that caused the `com.launchdarkly.client.files` package (the test file data source component added in v4.5.0) to be inaccessible unless you were using the "thin" jar.
- Stream connection errors are now logged at `WARN` level, rather than `ERROR`.

## [4.5.0] - 2018-10-26
### Added:
It is now possible to inject feature flags into the client from local JSON or YAML files, replacing the normal LaunchDarkly connection. This would typically be for testing purposes. See `com.launchdarkly.client.files.FileComponents`.

## [4.4.1] - 2018-10-15
### Fixed:
- The SDK's Maven releases had a `pom.xml` that mistakenly referenced dependencies that are actually bundled (with shading) inside of our jar, resulting in those dependencies being redundantly downloaded and included (without shading) in the runtime classpath, which could cause conflicts. This has been fixed. ([#122](https://github.com/launchdarkly/java-server-sdk/issues/122))

## [4.4.0] - 2018-10-01
### Added:
- The `allFlagsState()` method now accepts a new option, `FlagsStateOption.DETAILS_ONLY_FOR_TRACKED_FLAGS`, which reduces the size of the JSON representation of the flag state by omitting some metadata. Specifically, it omits any data that is normally used for generating detailed evaluation events if a flag does not have event tracking or debugging turned on.

### Fixed:
- JSON data from `allFlagsState()` is now slightly smaller even if you do not use the new option described above, because it completely omits the flag property for event tracking unless that property is `true`.

## [4.3.2] - 2018-09-11
### Fixed:
- Event delivery now works correctly when the events are being forwarded through a [LaunchDarkly Relay Proxy](https://github.com/launchdarkly/ld-relay).


## [4.3.1] - 2018-09-04
### Fixed:
- When evaluating a prerequisite feature flag, the analytics event for the evaluation did not include the result value if the prerequisite flag was off.
- The default Gson serialization for `LDUser` now includes all user properties. Previously, it omitted `privateAttributeNames`.

## [4.3.0] - 2018-08-27
### Added:
- The new `LDClient` method `allFlagsState()` should be used instead of `allFlags()` if you are passing flag data to the front end for use with the JavaScript SDK. It preserves some flag metadata that the front end requires in order to send analytics events correctly. Versions 2.5.0 and above of the JavaScript SDK are able to use this metadata, but the output of `allFlagsState()` will still work with older versions.
- The `allFlagsState()` method also allows you to select only client-side-enabled flags to pass to the front end, by using the option `FlagsStateOption.CLIENT_SIDE_ONLY`. ([#112](https://github.com/launchdarkly/java-server-sdk/issues/112))
- The new `LDClient` methods `boolVariationDetail`, `intVariationDetail`, `doubleVariationDetail`, `stringVariationDetail`, and `jsonVariationDetail` allow you to evaluate a feature flag (using the same parameters as you would for `boolVariation`, etc.) and receive more information about how the value was calculated. This information is returned in an `EvaluationDetail` object, which contains both the result value and an `EvaluationReason` which will tell you, for instance, if the user was individually targeted for the flag or was matched by one of the flag's rules, or if the flag returned the default value due to an error.

### Fixed:
- Fixed a bug in `LDUser.Builder` that would throw an exception if you initialized the builder by copying an existing user, and then tried to add a custom attribute.

### Deprecated:
- `LDClient.allFlags()`

## [4.2.2] - 2018-08-17
### Fixed:
- When logging errors related to the evaluation of a specific flag, the log message now always includes the flag key.
- Exception stacktraces are now logged only at DEBUG level. Previously, some were being logged at ERROR level.

## [4.2.1] - 2018-07-16
### Fixed:
- Should not permanently give up on posting events if the server returns a 400 error.
- Fixed a bug in the Redis store that caused an unnecessary extra Redis query (and a debug-level log message about updating a flag with the same version) after every update of a flag.

## [4.2.0] - 2018-06-26
### Added:
- New overloads of `LDUser.Builder.custom` and `LDUser.Builder.privateCustom` allow you to set a custom attribute value to any JSON element.

### Changed:
- The client now treats most HTTP 4xx errors as unrecoverable: that is, after receiving such an error, it will not make any more HTTP requests for the lifetime of the client instance, in effect taking the client offline. This is because such errors indicate either a configuration problem (invalid SDK key) or a bug, which is not likely to resolve without a restart or an upgrade. This does not apply if the error is 400, 408, 429, or any 5xx error.
- During initialization, if the client receives any of the unrecoverable errors described above, the client constructor will return immediately; previously it would continue waiting until a timeout. The `initialized()` method will return false in this case.

## [4.1.0] - 2018-05-15

### Added:
- The new user builder methods `customValues` and `privateCustomValues` allow you to add a custom user attribute with multiple JSON values of mixed types. ([#126](https://github.com/launchdarkly/java-server-sdk/issues/126))
- The new constant `VersionedDataKind.ALL` is a list of all existing `VersionedDataKind` instances. This is mainly useful if you are writing a custom `FeatureStore` implementation.

## [4.0.0] - 2018-05-10

### Changed:
- To reduce the network bandwidth used for analytics events, feature request events are now sent as counters rather than individual events, and user details are now sent only at intervals rather than in each event. These behaviors can be modified through the LaunchDarkly UI and with the new configuration option `inlineUsersInEvents`. For more details, see [Analytics Data Stream Reference](https://docs.launchdarkly.com/v2.0/docs/analytics-data-stream-reference).
- When sending analytics events, if there is a connection error or an HTTP 5xx response, the client will try to send the events again one more time after a one-second delay.
- The `LdClient` class is now `final`.

### Added:
- New methods on `LDConfig.Builder` (`updateProcessorFactory`, `featureStoreFactory`, `eventProcessorFactory`) allow you to specify different implementations of each of the main client subcomponents (receiving feature state, storing feature state, and sending analytics events) for testing or for any other purpose. The `Components` class provides factories for all built-in implementations of these.

### Deprecated:
- The `featureStore` configuration method is deprecated, replaced by the new factory-based mechanism described above.


## [3.0.3] - 2018-03-26
### Fixed
* In the Redis feature store, fixed a synchronization problem that could cause a feature flag update to be missed if several of them happened in rapid succession.
* Fixed a bug that would cause a `NullPointerException` when trying to evaluate a flag rule that contained an unknown operator type. This could happen if you started using some recently added feature flag functionality in the LaunchDarkly application but had not yet upgraded the SDK to a version that supports that feature. In this case, it should now simply treat that rule as a non-match.

### Changed
* The log message "Attempted to update ... with a version that is the same or older" has been downgraded from `WARN` level to `DEBUG`. It can happen frequently in normal operation when the client is in streaming mode, and is not a cause for concern.

## [3.0.2] - 2018-03-01
### Fixed
- Improved performance when evaluating flags with custom attributes, by avoiding an unnecessary caught exception (thanks, [rbalamohan](https://github.com/launchdarkly/java-server-sdk/issues/113)).


## [3.0.1] - 2018-02-22
### Added
- Support for a new LaunchDarkly feature: reusable user segments.

### Changed
- The `FeatureStore` interface has been changed to support user segment data as well as feature flags. Existing code that uses `InMemoryFeatureStore` or `RedisFeatureStore` should work as before, but custom feature store implementations will need to be updated.
- Removed deprecated methods.


## [3.0.0] - 2018-02-21

_This release was broken and should not be used._


## [2.6.1] - 2018-03-01
### Fixed
- Improved performance when evaluating flags with custom attributes, by avoiding an unnecessary caught exception (thanks, [rbalamohan](https://github.com/launchdarkly/java-server-sdk/issues/113)).


## [2.6.0] - 2018-02-12
## Added
- Adds support for a future LaunchDarkly feature, coming soon: semantic version user attributes.

## Changed
- It is now possible to compute rollouts based on an integer attribute of a user, not just a string attribute.


## [2.5.1] - 2018-01-31

## Changed
- All threads created by the client are now daemon threads. 
- Fixed a bug that could result in a previously deleted feature flag appearing to be available again.
- Reduced the logging level for use of an unknown feature flag from `WARN` to `INFO`.


## [2.5.0] - 2018-01-08
## Added
- Support for specifying [private user attributes](https://docs.launchdarkly.com/docs/private-user-attributes) in order to prevent user attributes from being sent in analytics events back to LaunchDarkly. See the `allAttributesPrivate` and `privateAttributeNames` methods on `LDConfig.Builder` as well as the `privateX` methods on `LDUser.Builder`.

## [2.4.0] - 2017-12-20
## Changed
- Added an option to disable sending analytics events
- No longer attempt to reconnect if a 401 response is received (this would indicate an invalid SDK key, so retrying won't help)
- Simplified logic to detect dropped stream connections
- Increased default polling interval to 30s
- Use flag data in redis before stream connection is established, if possible (See #107)
- Avoid creating HTTP cache when streaming mode is enabled (as it won't be useful). This makes it possible to use the SDK in Google App Engine and other environments with no mutable disk access.


## [2.3.4] - 2017-10-25
## Changed
- Removed GSON dependency from default jar (fixes #103)

## [2.3.2] - 2017-09-22
## Changed
- Only log a warning on the first event that overflows the event buffer [#102]

## [2.3.1] - 2017-08-11
## Changed
- Updated okhttp-eventsource dependency to 1.5.0 to pick up better connection timeout handling.

## [2.3.0] - 2017-07-10
## Added
- LDUser `Builder` constructor which accepts a previously built user as an initialization parameter.


## [2.2.6] - 2017-06-16
## Added
- #96 `LDUser` now has `equals()` and `hashCode()` methods

## Changed
- #93 `LDClient` now releases resources more quickly when shutting down

## [2.2.5] - 2017-06-02
## Changed
- Improved Gson compatibility (added no-args constructors for classes we deserialize)
- Automated release process 


## [2.2.4] - 2017-06-02
## Changed
- Improved Gson compatibility (added no-args constructors for classes we deserialize)
- Automated release process 


## [2.2.3] - 2017-05-10
### Fixed
- Fixed issue where stream connection failed to fully establish

## [2.2.2] - 2017-05-05
### Fixed
- In Java 7, connections to LaunchDarkly are now possible using TLSv1.1 and/or TLSv1.2
- The order of SSE stream events is now preserved. ([launchdarkly/okhttp-eventsource#19](https://github.com/launchdarkly/okhttp-eventsource/issues/19))

## [2.2.1] - 2017-04-25
### Fixed
- [#92](https://github.com/launchdarkly/java-server-sdk/issues/92) Regex `matches` targeting rules now include the user if
a match is found anywhere in the attribute.  Before fixing this bug, the entire attribute needed to match the pattern.

## [2.2.0] - 2017-04-11
### Added
- Authentication for proxied http requests is now supported (Basic Auth only)

### Changed
- Improved Redis connection pool management.

## [2.1.0] - 2017-03-02
### Added
- LdClientInterface (and its implementation) have a new method: `boolean isFlagKnown(String featureKey)` which checks for a 
feature flag's existence. Thanks @yuv422!

## [2.0.11] - 2017-02-24
### Changed
- EventProcessor now respects the connect and socket timeouts configured with LDConfig.

## [2.0.10] - 2017-02-06
### Changed
- Updated okhttp-eventsource dependency to bring in newer okhttp dependency
- Added more verbose debug level logging when sending events

## [2.0.9] - 2017-01-24
### Changed
- StreamProcessor uses the proxy configuration specified by LDConfig.

## [2.0.8] - 2016-12-22
### Changed
- Better handling of null default values.

## [2.0.7] - 2016-12-21
### Changed
- allFlags() method on client no longer returns null when client is in offline mode.

## [2.0.6] - 2016-11-21
### Changed
- RedisFeatureStore: Update Jedis dependency. Improved thread/memory management.

## [2.0.5] - 2016-11-09
### Changed
- The StreamProcessor now listens for heartbeats from the streaming API, and will automatically reconnect if heartbeats are not received.

## [2.0.4] - 2016-10-12
### Changed
- Updated GSON dependency version to 2.7

## [2.0.3] - 2016-10-10
### Added
- StreamingProcessor now supports increasing retry delays with jitter. Addresses [https://github.com/launchdarkly/java-server-sdk/issues/74[(https://github.com/launchdarkly/java-server-sdk/issues/74)

## [2.0.2] - 2016-09-13
### Added
- Now publishing artifact with 'all' classifier that includes SLF4J for ColdFusion or other systems that need it.

## [2.0.1] - 2016-08-12
### Removed
- Removed slf4j from default artifact: [#71](https://github.com/launchdarkly/java-server-sdk/issues/71)

## [2.0.0] - 2016-08-08
### Added
- Support for multivariate feature flags. New methods `boolVariation`, `jsonVariation` and `intVariation` and `doubleVariation` for multivariates.
- Added `LDClientInterface`, an interface suitable for mocking `LDClient`.

### Changed
- The `Feature` data model has been replaced with `FeatureFlag`. `FeatureFlag` is not generic.
- The `allFlags` method now returns a `Map<String, JsonElement>` to support multivariate flags.

### Deprecated
- The `toggle` call has been deprecated in favor of `boolVariation`.

### Removed
- The `getFlag` call has been removed.
- The `debugStreaming` configuration option has been removed.
