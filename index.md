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

### Endpoints

HTTP-RPC uses normal URL endpoints, with paths that follow the schema

![URl schema](/http-rpc-spec/assets/img/url-schema.svg){:class="img-responsive"}

for example `https://example.com/billing/invoice.send`.

The `namespace` represents a logical grouping or domain of the contained resources.

A `resource` is an individual domain object, such as `user`, `invoice` or `project`.

`action` is an operation that can be performed on the specified resource. It must be in verb form, such as `create`, `get`, `send`, `import`, `validate` etc.

`namespace`, `resource` and `action` are limited to lower-case alphanumeric characters and `_` and must start with an alphanumeric character (`[a-z][a-z0-9_]+`).

#### Methods

HTTP-RPC endpoint must support the `GET` and `POST` request methods.

##### GET

A `GET` request must return the Avro schema for the endpoint, with the schema outlined on the section `Endpoint Schema`

##### POST

A `POST` request corresponds to execution of the endpoint action. It takes an Avro payload of the form outlined on the [Request Payload schema](#request-payload-schema) section and returns an answer outlined in the [Response Payload schema](#response-payload-schema) section.

The HTTP Response code must always be 200.

### Schemas

#### Request Payload Schema

#### Response Payload Schema

Responses must follow the schema

```json
{
    "type": "record",
    "name": "Response",
    "fields": [
        {
            "name": "result",
            "type": [
                {
                    "type": "record",
                    "name": "Result",
                    "doc": "Add type according to endpoint with matching fields",
                    "fields": []
                }
                , "null"]
        },
        {
            "name": "error",
            "type": ["null",
                {
                    "type": "record",
                    "name": "Error",
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

The `Result` type above is a placeholder to be substituted with your actual response schema.

The `identifier` on the `Error` record should be a short string error code to support automated handling of errors, e.g. `file_not_found`. The `description` field should contain a human-readable error message for consumption by users.

`additionalInformation` can be filled freely with more context about an error, such as stack traces.
