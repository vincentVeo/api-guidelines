# REST API Guidelines

These API guidelines are used to guide design of the IBM's Watson Developer Cloud services, but may provide insight for other REST APIs as well.

> We also have a [code-style](code-style.md) and [repository guidelines](repository-guidelines.md), take a look 😎

## Table of Content

- [REST API Guidelines](#rest-api-guidelines)
	- [Naming](#naming)
	- [JSON vs CSV vs XML](#json-vs-csv-vs-xml)
	- [JSON Structure](#json-structure)
		- [Avoid Anonymous Arrays](#avoid-anonymous-arrays)
		- [Avoid Dynamic Keys](#avoid-dynamic-keys)
		- [Pretty-Printed Responses](#pretty-printed-responses)
	- [Multiple Languages](#multiple-languages)
	- [User-Created Resources](#user-created-resources)
		- [Resource IDs](#resource-ids)
	- [Asynchronous Operations](#asynchronous-operations)
	- [Multi-lingual Support](#multi-lingual-support)
	- [Dates](#dates)
	- [Errors](#errors)
		- [Error Status Codes](#error-status-codes)
		- [Error JSON Objects](#error-json-objects)
	- [`GET` vs `POST` vs `PUT`](#get-vs-post-vs-put)
	- [Common API Problems](#common-api-problems)
		- [`GET` vs `POST` vs `PUT`](#get-vs-post-vs-put)
		- [Quoted numbers](#quoted-numbers)
		- [Naming](#naming)
		- [Anonymous JSON arrays](#anonymous-json-arrays)
	- [Watson Developer Cloud Guidelines](#watson-developer-cloud-guidelines)
		- [Versioning](#versioning)
		- [Bluemix Lifecycle](#bluemix-lifecycle)
		- [Metadata](#metadata)
		- [URL Basepaths](#url-basepaths)
	- [Other API Design Resources](#other-api-design-resources)


## Naming

The words used should match the users' terminology.

All names specified in path and query parameters, resource names, and JSON input and output fields and pre-defined values must use "snake_case" and must not use abbreviations or acronyms.

Good:

* `classifier_id`
* `status_description`
* `generate_visualization`

Bad:

* `observeResult`
* `lang`
* `DateRange`
* `vad_score`

Exceptions (cases when we use abbreviations or acronyms):

* `en`, `en-us`. (Since there's an established convention for language codes, we use these.)
* `ibm`, `http`, etc. (When an acronym or abbreviation is more common than the full name, we use the shorter name instead.)

## JSON vs CSV vs XML

APIs should support JSON for non-binary data input and output, and may also support CSV or XML when desired (although it's not usually recommended), using the `Content-Type` (for input) and `Accept` (for output) headers to specify formats. Formats should default to JSON when not specified.

## JSON Structure

### Avoid Anonymous Arrays

For JSON requests and responses, APIs should not use top-level, anonymous arrays. Wrapping the arrays in a JSON object makes it possible to add additional fields later.

Good:

* `{"items": ["item 1", "item 2"]}`

Bad:

* `["item 1", "item 2"]`

### Avoid Dynamic Keys

The keys in JSON objects should be static strings, not dynamic user-provided values. (Parsing a JSON object that represents a hash-map of user-provided keys and values is difficult in JavaScript, requiring the filtering out of native properties using `hasOwnProperty()`.)

Good:

* `{"user_key": "my user key", "user_value": "my user value"}`

Bad:

* `{"my user key": "my user value"}`

### Pretty-Printed Responses

Results should be pretty printed by default. The data overhead, with gzip enabled, is negligible. [http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api#pretty-print-gzip](http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api#pretty-print-gzip)

## Multiple Languages

When APIs support content in multiple languages (English or Spanish, for example), the language should be specified using the `Content-Language` header field, with language values following [RFC 1766](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.10), for example "en" or "es-ES".

When handling input data where multiple parts of the input can be in multiple languages, a JSON field of "language" can be used (overriding the header field, if it exists), for example: `{"profiles": [{"text": "hello", "language": "en"}, {"text": "hola", "language": "es"}]}`

## User-Created Resources

User-created resources should be created by `POST`ing to /*resource_name*, and accessed at /*resource_name*/*resource_id*, where *resource_id* is generated by the service.

### Resource IDs

User-created resources IDs should be generated by the service (not specified by the users), and be globally unique (not just unique to the user). For a friendly, non-unique identifier, resources should include a user-specified "name" field.

IDs can be GUIDs (`9fd3b33a-5b7d-4e79-9bca-96054e3556f8`), or shorter unique strings (`47C164-nlc-243`), and can optionally include a truncated, sanitized version of the user-specified "name" field to make the id more readable (i.e. `47C164243-my_weather_classifier`).

## Asynchronous Operations

For operations that take more than ~15 seconds, users should initial an operation with a `POST`, which should return the ID of the resource for the newly created resource / asynchronous job.

For example:

`POST` `https://gateway.watsonplatform.net/natural-language-classifier/classifiers` responds with:

```
{
  "classifier_id": "47C164-nlc-243",
  "name": "weather",
  "language": "en",
  "created": "2015-08-24T18:42:25.324Z",
  "url": "https://gateway.watsonplatform.net/natural-language-classifier/api/v1/classifiers/47C164-nlc-243",
  "status": "Training",
  "status_description": "The classifier instance is in its training phase, not yet ready to accept classify requests"
}
```

To retrieve the status of the operation, or the results (when the operation completes), the same url should be used, for example:

`GET` `https://gateway.watsonplatform.net/natural-language-classifier/api/v1/classifiers/47C164-nlc-243`

If the asychronous operation results are meant to be ephemeral, instead of a persistent resource, then users should delete the resource after retrieval:

`DELETE` `https://gateway.watsonplatform.net/natural-language-classifier/api/v1/classifiers/47C164-nlc-243`

### External input and output

For very large inputs or outputs, a service may also accept Object Storage pointers and credentials to asynchronously read in very large input and write out very large output.

## Multi-lingual Support

Language codes should be specified in input and output as "en", "en-US". Left unspecified the language should default to "en".

Language codes without country dialects ("en", "es") should use a default dialect for that language ("en-US", "es-ES"), if there are multiple dialects available or the single available language model is dialect specific. The response should include the language that was used, separate from the language that was specified. (Exception: Chinese should require a dialect to differentiate between "zh-CN" and "zh-TW", and not provide a default for "zh".)

Dialects that don't have a direct match should match more generic or default models, when the models used are marked as supporting the whole language code. For example, "en-GB" could match an "en" or "en-US" model that was marked as supporting en-\*.

The standard http header [content-language](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.12) should be supported to specify the input language.

A "language" field in JSON input can also be supported, and should take precedence over the header. The JSON "language" field may apply to individual parts of input, while the header provides a default language value for all of the input in the request.

## Dates

Dates returned in responses should be formatted using ISO 8601 like: `yyyy-MM-dd'T'HH:mm:ss.SSS'Z'`, for example `2015-08-24T18:42:25.324Z`.  All date/times returned in responses should be in the UTC time zone (that is why the 'Z' is required literally).

Date taken as input should be formatted the same, with the seconds and time fields optional (for example: `2015-08-24` or `2015-08-24T18:42`). Date / times taken as input should be flexible in the time zones they accept (any valid time zone should be accepted), for example: `2015-08-24T18:42:25.324-0400`.

## Errors

### Error Status Codes

5xx errors should be *only* for server errors and uptime failures &mdash; there should be no repeatable way for users to generate 5xx error codes. 4xx errors should be used when the failure is a result of an incorrect or unsupported user input.

[W3 HTTP status codes](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)

### Error JSON Objects

Examples:

* `{"code": 404, "error": "Not found"}`
* `{"code": 400, "error": "Missing text", "description": "The required 'text' parameter is missing."}`

## `GET` vs `POST` vs `PUT`

`GET` methods should be safe &mdash; they should not have side effects &mdash; and [idempotent](http://restcookbook.com/HTTP%20Methods/idempotency/) (they should return the same result until the underlying resource changes).

`POST` methods need not be safe or idempotent, but should avoid query parameters. Methods that are safe should use `GET` instead, unless parameters larger than ~4KB are required, in which case equivalent `GET` and `POST`

`PUT` methods should be idempotent (calling the `PUT` method multiple times with the same parameters should be indistinguishable from calling it once) and should fully replace the resource and all sub-resources. `PUT` should also avoid query parameters.

Note: We've avoided `PATCH` so far because of the two competing `PATCH` formats, JSON Patch violates multiple of our other API guidelines (top-level arrays, abbreviations), while JSON Merge lacks the expressiveness to remove fields.

## Common API Problems

### `GET` vs `POST` vs `PUT`

Are all `GET` methods safe, and do `POST` calls need to be `POST`?

### Quoted numbers

Are any of the numeric fields accidentally returned as strings? (`"value": "0.923"` instead of `"value": 0.923`)

### Naming

Do all identifiers (resource names, parameter names, JSON field names) use snake_case and not use acronyms or abbreviations?

### Anonymous JSON arrays

When input or output JSON structures contain top level anonymous arrays, for example: `["item 1", "item 2"]` instead of `{"items": ["item 1", "item 2"]}` they are harder to extend later.

## Watson Developer Cloud Guidelines

(This part of the guidelines are only applicable to IBM Watson's Developer Cloud Services.)

### Versioning

Services should include a major version in the path (`/v1/`) and a minor version as a required query parameter that takes a date: `?version=2015-11-24` (inspired by the [Foursquare](https://developer.foursquare.com/overview/versioning) model of date-versioning). This minor version can then be used to control the behavior of minor breaking changes, so that when an output field name, status code, or other change is made users are not affected until they update their version. (Changes to underlying models are not considered breaking and are not controlled by the version parameter.)

Developers should never pass the current date as a version, and should instead use the latest version available when writing code and periodically check for changes and update their code and version to take advantage of them.

#### What counts as "breaking"

At a high level, a "breaking" change is a change that would cause "reasonable" code to stop working. What's "reasonable" is subjective, but examples of changes we consider non-breaking include:

* Addition of output fields
* Addition of (optional) input parameters
* Changes to underlying models and algorithms that may result in different results and values
* Changes to string values (in most cases; dates and other structured strings are exceptions to this)

Changes we consider breaking include:

* Removing output fields
* Addition of a required input parameter
* Changes to parameter default values
* Change of field names
* Change of status codes (ideally; historically we've made changes between 4xx and other 4xx, and 2xx and other 2xx, without considering them breaking)

#### Date versioning FAQ

##### Why make the version date required? Wouldn't it be simpler to make it optional?

The main reason not to provide a default version is that the only stable behavior would have to be defaulting to the *oldest* version. We want developers to target the newest version at the date they write their code, but in a way that won't break their code when we make breaking changes. (FourSquare stopped accepting requests without version dates, likely for the same reasons.)

The exception is that when adding version dates to an existing public API major version that did not previously have version dates, the introduction of the version parameter would have to be optional to avoid being a breaking change.

##### My service would never use this, since it would be too expensive to maintain multiple versions.

One of the most common of these changes are design defects that slipped unnoticed into a public release, such as accidental quotation of numbers, unnecessary output fields, field names that don't follow the API's naming patterns, status codes that don't follow the REST conventions (especially in error scenarios), incorrect date formats, error response field names that match the rest of the API and platform, and default parameter values that cause usability or performance problems. (Each of these issues has occurred at least once in public Watson Developer Cloud APIs.) Stripe's [API changelog](https://stripe.com/docs/upgrades#api-changelog) has more examples of the types of API fixes and evolutions that are useful to be able to make.

In the absence of version dates, these defects persist in the API until the next major version; with version dates, they can be addressed for new users without breaking early adopters' code. With version dating, a single major version of the service can conditionally follow the new API design. Even if a service doesn't intend to make breaking changes, historically the need for them has been common.

##### Should a service reject calls where the version date matches the current date or a future date?

The current answer to this is no, although the reasons for this are less strong than those around making version dates required, and are more implementation centric than API centric. One reason to not reject requests for future dates is to make it easier to test and deploy features that will be enabled in a future release; internal testing of a new change that will take effect at a future date is simpler when there isn't code (or production-conditioned code) that compares the version date to the current date.

Rejecting the current date is even less desirable, since it might be a valid date, and the developer pattern of targeting the API by given the date the code was written is valid. From a service implementation standpoint, not comparing the version date parameter is the fastest and simplest strategy. From a users' standpoint, however, the behavior of passing a future version date should be considered undefined, since a new release with breaking changes might be released (or we may switch to rejecting future version dates).

### Bluemix Lifecycle

Provisioning or binding a service in Bluemix should not automatically create resources within the service, and users shouldn't need to create two instances of a service on Bluemix to obtain more resources. Instead, creation and deletion of resources should be through REST APIs. (Deprovisioning a service should remove any resources that are no longer accessible.)

### Metadata

To allow developers to keep track of metadata associated with their REST calls (like customer ids, testing flags), services should accept the header `X-Watson-Metadata`, and echo this as a response header. Developers are encouraged to use semi-colon separated values and name=value pairs, such as: `X-Watson-Metadata: customer_id=1242;testing;platform=iOS`

### URL Basepaths

Services should follow the pattern:

https://gateway.watsonplatform.net/*full-service-name-release*/api/v1

Where *-release* is *-experimental*, *-beta*. (For GA releases this is omitted.)

For example:

https://gateway.watsonplatform.net/tone-analyzer-experimental/api/  
https://gateway.watsonplatform.net/personality-insights-beta/api/  
https://gateway.watsonplatform.net/language-translation/api/  


## Other API Design Resources

[Best Practices for Designing a Pragmatic RESTful API](http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api) - Overview of basic REST API design

[18F API Standards](https://github.com/18f/api-standards) - REST API guidelines from Whitehouse technology team

[Move Fast, Don't Break Your API](http://www.heavybit.com/library/video/2014-09-30-amber-feng) - Talk about Stripe's use of version dates
