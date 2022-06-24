## HTTP-RPC Specification

### Introduction

This is the specification for HTTP-RPC, an architectural style for modeling web services in a flexible, scalable, portable, simple and realiable way. HTTP-RPC is easy for humans to understand and develop and easy for machines to parse and interact with. Its aim is to capture and formalize approaches to web services commonly encountered in the wild and that web services organically tend to evolve towards over time, but in a structured and maintainable way.

At its core, HTTP-RPC defines how endpoints should be structured and how to interact with them and with each other, how errors should be handled, how data exchange happens and how this get documented.

### Overview
HTTP-RPC is a standard for creating resource-oriented service endpoints that support RPC style interaction. It defines how these endpoints should be structured and organized, the base schemas to use in communication, how errors are handled and how these services can be self-describing and machine consumable.

### Data Exchange Format

HTTP-RPC uses [Avro](https://avro.apache.org/) as its data exchange format. Avro itself already defines ways of RPC communication, but HTTP-RPC aims to fit the context of modern service-oriented architectures better than a single-point RPC service like Avros can. It also tries to cover use-cases that are left unspecified in an RPC specification like Avro.

Avro is used as the data exchange format because it fully supports plain JSON, allowing developers to use existing tools and knowledge, but also supports its heavily optimized binary format that can greatly improve performance and reduce bandwidth. It's support for type safety, schemas and schema evolution also adresses many issues that can be seen in modern webservices, making it a great fit for the purposes of HTTP-RPC.
