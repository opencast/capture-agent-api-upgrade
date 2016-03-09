Things that have already been decided
-------------------------------------

*#proposal by Greg: CA API #proposal (Closes 2015-01-30T15:00)*


How to document changes

 - Discussion should happen on list
 - Decision summaries should be sent to vendor list

How to deprecate outdated functionality

 - Two CA-API version support
 - Parts that are deprecated in CA-API version x may be removed in x+2

Versioning

 - Available versions are listed at a /services/versions.{format} endpoint
	which returns a blob of metadata that looks like:

        versions {
          available: [ "v1.0.0", "v1.1.0" ]
          default: "v1.1.0"
        }

 - Requests may specify a specific version in their `HTTP ACCEPT` header.  For example:

        application/vnd.opencast.capture.1.0.0+xml


 - Responses specify the version in their Content-Type header
 - Unspecified versions will default to the default version as advertised by
	the versions endpoint.
 - Unknown or unparsable versions lead to a 400 (bad request) response
