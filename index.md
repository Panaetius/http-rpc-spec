---
layout: default
title: HTTP-RPC Specification (Draft)
---


## HTTP-RPC Specification (Draft)

* toc
{:toc}
### Introduction

This is the specification for HTTP-RPC, an architectural style for modeling web services in a flexible, scalable, portable, simple and realiable way. HTTP-RPC is easy for humans to understand and develop and easy for machines to parse and interact with. Its aim is to capture and formalize approaches to web services commonly encountered in the wild and that web services organically tend to evolve towards over time, but in a structured and maintainable way.

At its core, HTTP-RPC defines how endpoints should be structured and how to interact with them and with each other, how errors should be handled, how data exchange happens and how this get documented.

### Overview

HTTP-RPC is a standard for creating resource-oriented service endpoints that
support RPC style interaction. It defines how these endpoints should be
structured and organized, the base schemas to use in communication, how errors
are handled and how these services can be self-describing and machine
consumable. The aim is to combine the best of both worlds, being able to draw on
the myriad of tool available in the HTTP/REST while benefitting from the
flexibility that RPC offers, but limiting how it can be implemented to encourage
implementations to stay as homogeneous as possible, improving portability over
existing RPC approaches.

### Data Exchange Format

HTTP-RPC uses [Avro](https://avro.apache.org/) as its data exchange format. Avro itself already defines ways of RPC communication, but HTTP-RPC aims to fit the context of modern service-oriented architectures better than a single-point RPC service like Avros can. It also tries to cover use-cases that are left unspecified in an RPC specification like Avro.

Avro is used as the data exchange format because it fully supports plain JSON, allowing developers to use existing tools and knowledge, but also supports its heavily optimized binary format that can greatly improve performance and reduce bandwidth. It's support for type safety, schemas and schema evolution also adresses many issues that can be seen in modern webservices, making it a great fit for the purposes of HTTP-RPC.

### Resources

HTTP-RPC uses normal URL endpoints, with paths that follow the schema

![URl schema](/http-rpc-spec/assets/img/url-schema.svg){:class="img-responsive"}

for example `https://example.com/billing/invoice.send`.

The `namespace` represents a logical grouping or domain of the contained resources.

A `resource` is an individual domain object, such as `user`, `invoice` or `project`. It can be just about anything that can be named.

`action` is an operation that can be performed on the specified resource. It must be in verb form, such as `create`, `get`, `send`, `import`, `validate` etc. All actions related to a resource should be grouped under that resource.

`namespace`, `resource` and `action` are limited to lower-case alphanumeric characters and `_` and must start with an alphanumeric character (`[a-z][a-z0-9_]+`).

#### Resource Methods

HTTP-RPC endpoints must support the `GET` and `POST` request methods.

##### POST Method

All requests for interacting with resources have to be done through the `POST` HTTP method.

A `POST` requests correspond to execution of the endpoint action. It takes an Avro payload as POST-body and returns a response as outlinedbelow.

The HTTP Response code must always be 200.

Responses must follow the schema

```json
{
    "name": "Response",
    "type": "record",
    "fields": [
        {
            "name": "result",
            "type": [
                {
                    "name": "...",
                    "type": "record",
                    "doc": "Add type according to endpoint with matching fields",
                    "fields": []
                }
                , "null"]
        },
        {
            "name": "error",
            "type": ["null",
                {
                    "name": "Error",
                    "type": "record",
                    "fields":[
                        {
                            "name": "identifier",
                            "type": "string"
                        },
                        {
                            "name": "description",
                            "type": "string"
                        },
                        {
                            "name": "additionalInformation",
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

The type in `result` is a placeholder to be substituted with your actual response type.

The `Error` type is the standarized error response, making it easy to process errors automatically as well as being useful to users.

The `identifier` on the `Error` record should be a short string error code to
support automated handling of errors, e.g. `file_not_found`. Error codes must be
unique to the error raised.

The `description` field should contain a human-readable error message for consumption by users.

`additionalInformation` can be filled freely with more context about an error, such as stack traces or translated error messages.

##### GET Method

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

#### Self-Descriptive

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
