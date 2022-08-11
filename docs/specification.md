---
title: HTTP-RPC Specification (Draft)
---
## Introduction

This is the specification for HTTP-RPC, an architectural style for modeling web services in a flexible, scalable, portable, simple and realiable way. HTTP-RPC is easy for humans to understand and develop and easy for machines to parse and interact with. Its aim is to capture and formalize approaches to web services commonly encountered in the wild and that web services organically tend to evolve towards over time, but in a structured and maintainable way.

At its core, HTTP-RPC defines how endpoints should be structured and how to interact with them and with each other, how errors should be handled, how data exchange happens and how this get documented.

## Overview

HTTP-RPC is a standard for creating resource-oriented service endpoints that
support RPC style interaction. It defines how these endpoints should be
structured and organized, the base schemas to use in communication, how errors
are handled and how these services can be self-describing and machine
consumable. The aim is to combine the best of both worlds, being able to draw on
the myriad of tool available in the HTTP/REST while benefitting from the
flexibility that RPC offers, but limiting how it can be implemented to encourage
implementations to stay as homogeneous as possible, improving portability over
existing RPC approaches.

## Data Exchange Format

HTTP-RPC uses [Avro](https://avro.apache.org/) as its data exchange format. Avro itself already defines ways of RPC communication, but HTTP-RPC aims to fit the context of modern service-oriented architectures better than a single-point RPC service like Avros can. It also tries to cover use-cases that are left unspecified in an RPC specification like Avro.

Avro is used as the data exchange format because it fully supports plain JSON, allowing developers to use existing tools and knowledge, but also supports its heavily optimized binary format that can greatly improve performance and reduce bandwidth. It's support for type safety, schemas and schema evolution also adresses many issues that can be seen in modern webservices, making it a great fit for the purposes of HTTP-RPC.


## HTTP/2

All HTTP-RPC servers must support HTTP/2. This allows a single connection to be used for all requests, in parallel, due to HTTP/2 multiplexing as well as allowing binary data to be sent or streamed more efficiently. HTTP/1.1 can be offered as a fallback if a client doesn't support HTTP/2, but is not strictly required.

### Streaming

In addition to normal request-response  communication, HTTP-RPC also supports streaming of requests/responses using HTTP/2 streams. Request streams, response streams and bidirectional streaming are supported.

### Headers

Most HTTP Request headers should work as normal with HTTP-RPC. Headers starting with `http-rpc-` are reserved for future use by the protocol.

Namely, you can use `Authorization` header for authentication, e.g. using OAuth2, `Access-Control-Allow-Origin` for Cross-origin Resource Sharing (CORS), `Cache-Control` for HTTP caching, `Set-Cookie`/`Cookie` for HTTP cookies, `Location` for redirects and so on.

Similarily, HTTP Response headers should also work as expected.

#### HTTP-RPC Specific Request Headers

- **http-rpc-timeout**: Timeout for the request, after which the server must cancel it and return an error response. Positive integer followed by a unit, e.g. `15s`. Allowed units are `H`(hour), `M`(minute), `s`(second), `m`(millisecond).
- **http-rpc-compression**: Compression to use in the response, defaults to `none`. Possible values are `none`(No Compression), `gzip` (GZip compression), `zstd` (ZStandard compression), `snappy` (snappy compression). Custom compressions can also be added and supported by implementations.

##### HTTP-RPC Specific Response Headers

- **http-rpc-compression**: Compression used in the response. Possible values are `none`(No Compression), `gzip` (GZip compression), `zstd` (ZStandard compression), `snappy` (snappy compression). Custom compressions can also be added and supported by implementations.

## Resources

HTTP-RPC uses normal URL endpoints, with paths that follow the schema

![URl schema](/assets/img/url-schema.svg)

for example `https://example.com/billing/invoice.send`.

The `namespace` represents a logical grouping or domain of the contained resources.

A `resource` is an individual domain object, such as `user`, `invoice` or `project`. It can be just about anything that can be named.

`action` is an operation that can be performed on the specified resource. It must be in verb form, such as `create`, `get`, `send`, `import`, `validate` etc. All actions related to a resource should be grouped under that resource.

`namespace`, `resource` and `action` are limited to lower-case alphanumeric characters and `_` and must start with an alphanumeric character (`[a-z][a-z0-9_]+`).

### Resource Methods

HTTP-RPC endpoints must support the `GET` and `POST` request methods.

#### POST Method

All requests for interacting with resources have to be done through the `POST` HTTP method.

A `POST` requests correspond to execution of the endpoint action. It takes an Avro payload as POST-body and returns a response as outlinedbelow.

The HTTP Response code must always be 200.

Responses must follow the schema

```json
{
    "name": "Response", // (1)
    "type": "record",
    "fields": [
        {
            "name": "result", // (2)
            "type": [
                {
                    "name": "...", // (3)
                    "type": "record",
                    "doc": "Add type according to endpoint with matching fields", // (4)
                    "fields": []
                }
                , "null"]
        },
        {
            "name": "error", // (5)
            "type": ["null",
                {
                    "name": "Error", // (6)
                    "type": "record",
                    "fields":[
                        {
                            "name": "identifier", // (7)
                            "type": "string"
                        },
                        {
                            "name": "description", // (8)
                            "type": "string"
                        },
                        {
                            "name": "additionalInformation", // (9)
                            "type": "map",
                            "values": "string"
                        }
                    ]
                }
            ]
        }
    ]
}
```

1. All response schemas are of type `Response`
2. The result of a request, if everything went well
3. Name this type to whatever type your request returns and fill in the schema with the relevant fields
4. The doc string is mandatory and is used to describe the endpoint functionality in human readable form
5. The error field can be empty, if everything worked or contain an error record if something failed in the application code
6. All errors must follow the error schema
7. The error identifier should be an alphanumeric string that allows handling the error by code
8. The description contains human readable text about the error, why it happened and how it could be fixed, if possible
9. this field allows sending additional context for the error, such as stacktraces, translations of the error message or further context

The type in `result` is a placeholder to be substituted with your actual response type.

The `Error` type is the standarized error response, making it easy to process errors automatically as well as being useful to users.

The `identifier` on the `Error` record should be a short string error code to
support automated handling of errors, e.g. `file_not_found`. Error codes must be
unique to the error raised.

The `description` field should contain a human-readable error message for consumption by users.

`additionalInformation` can be filled freely with more context about an error, such as stack traces or translated error messages.

#### GET Method

A `GET` request must return the Avro schema for the endpoint, with the schema outlined in the section [Self-Descriptive](#self-descriptive). This allows clients to get all necessary information about the resource and its action dynamically and can also be used to verify responses from the service.

Responses must follow the schema:

```json
{
    "name": "ActionSchema",
    "type": "record",
    "fields": [
        {
            "name": "namespace",
            "type": "string"
        },
        {
            "name": "resource",
            "type": "string"
        },
        {
            "name": "action",
            "type": "string"
        },
        {
            "name": "description",
            "type": "string"
        },
        {
            "name": "RequestSchema",
            "type": {
                "name": "...",
                "type": "record",
                "doc": "Add type according to endpoint request",
                "fields": []
            }
        },
        {
            "name": "ResponseSchema",
            "type": {
                "name": "...",
                "type": "record",
                "doc": "Add type according to endpoint response",
                "fields": []
            }
        }
    ]
}
```

### Self-Descriptive

HTTP-RPC services should be completely self-descriptive. All a client has to know is the base URL of the service.

Doing a `GET` request on any action endpoint returns the request and response schemas for that endpoint.

Doing a `GET` request on the base URL returns a description of all namespaces, their resources and their actions. The base URL endpoint must respond with the following schema:

``` json
{
    "name": "ServiceSchema",
    "type": "record",
    "fields": [
        {
            "name": "namespaces",
            "type": "array",
            "items": {
                "name": "NamespaceSchema",
                "type": "record",
                "fields": [
                    {
                        "name": "namespace",
                        "type": "string"
                    },
                    {
                        "name": "description",
                        "type": "string"
                    },
                    {
                        "name": "resources",
                        "type": "array",
                        "items": {
                            "name": "ResourceSchema",
                            "type": "record",
                            "fields": [
                                {
                                    "name": "resource",
                                    "type": "string"
                                },
                                {
                                    "name": "description",
                                    "type": "string"
                                },
                                {
                                    "name": "actions",
                                    "type": "array",
                                    "items": "ActionSchema"
                                }
                            ]
                        }
                    }
                ]
            }
        }
    ]
}
```


### Interoperability with plain HTTP

Sometimes it might be neccessary to break with convention and have different content types and endpoints that don't follow the HTTP-RPC schema. For instance, client libraries that use `Content-Type: multipart/form-data` to upload files or endpoints that serve files directly using their respective MIME type. Ideally, these types of requests should still use HTTP-RPC and send data as binary in the Request/Response bodies and use of the interoperability feature is discouraged.

But for interoperability with existing libraries and browser functionality, HTTP-RPC allows mixing plain HTTP endpoints with HTTP-RPC endpoints. These types of endpoints need to be clearly distinguished from HTTP-RPC endpoints to avoid confusion. To this end, these endpoints must have routes like `<namespace>/external/<path>`, where `<path>` can be an arbitrary path.

These endpoints should still be specified in the main service schema:

``` json
{
    "name": "ServiceSchema",
    "type": "record",
    "fields": [
        {
            "name": "namespaces",
            "type": "array",
            "items": {
                "name": "NamespaceSchema",
                "type": "record",
                "fields": [
                    {
                        "name": "namespace",
                        "type": "string"
                    },
                    {
                        "name": "description",
                        "type": "string"
                    },
                    {
                        "name": "external_resources",
                        "type": "array",
                        "items": {
                            "name": "ExternalResourceSchema",
                            "type": "record",
                            "fields": [
                                {
                                    "name": "path",
                                    "type": "string"
                                },
                                {
                                    "name": "description",
                                    "type": "string"
                                }
                            ]
                        }
                    }
                ]
            }
        }
    ]
}
```

HTTP-RPC implementation are free to handle these plain HTTP endpoints themselves, for example by delegating to an underlying framework, or to forward those requests to another server implementation. None of the HTTP-RPC specific rules apply to these endpoints, other than the rule for the path format and the specification in the schema.