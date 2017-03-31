---
name: Configuration Management Extension
shortname: 5/CMX
status: raw
editor: Dmitry Sergeev <dsergeev@cybervisiontech.com>
---

## Introduction

The Configuration Management Extension is a [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

It is intended to manage endpoint configuration distribution.

## Requirements and constraints
### Problems and possible solutions

1. _Multiple configurations available for client._ It is possible that there will be more than one available configurations for endpoint (e.g., endpoint was offline while new configurations were added).
   
   Solution:
   - Only latest configuration should be sent to an endpoint to reduce amount of traffic.

2. _The server should know if a configuration has been delivered._ 

   Solutions:
   - Use request/response messaging pattern from [Kaa Protocol RFC1](/0001-kaa-protocol/README.md).

## Use cases

### UC1: pull
Configuration delivery by request. The endpoint should be able to receive latest configuration from CMX by request.

### UC2: push
Configuration delivery that is initiated by the server. The endpoint should receive latest configuration when it connects to a server if this configuration hadn't applied yet.

## Design

### Request/response
This topic is covered in [Kaa Protocol RFC1](/0001-kaa-protocol/README.md)


### Formats
#### Schemeless JSON
##### Configuration pull
Endpoint must send messages to the next topic in order to obtain configuration:
```
<endpoint_token>/pull/json
```

The payload should be a JSON-encoded object with the following fields:
- `id` (required) - id of the message. Should be either string or number. Used to match server response to the request.
- `configVersion` (optional) - current endpoint configuration version. Should be of integer type.
- `requiredConfigVersion` (optional) - endpoint configuration version that is requested by endpoint. This provides ability to re-send configuration for endpoints that cannot persist the configuration and provides mechanism to request previous configuration versions (roll-back).
If there's no `requiredConfigVersion` field in message, then server should use `configVersion` to determine is it needed to send response with configuration, server must not send configuration record if latest available configuration version equals to the one sent by endpoint. If `configVersion` field was excluded from the message, then server will send configuration anyway

Example:
```json
{
  "id": 42,
  "configVersion": 1,
  "requiredConfigVersion": 2
}
```
Example 2:
```json
{
  "id": 42,
  "configVersion": 3,
  "requiredConfigVersion": 2
}
```
Example 3:
```json
{
  "id": 42,
  "configVersion": 1,
}
```
Example 4:
```json
{
  "id": 42
}
```

A server response is a JSON record that is sent to the next topic:
```
 <endpoint_token>/pull/json/status
``` 
Message format:
- `id` (required) a copy of the `id` field from the corresponding request.
- `configVersion` (required) - version of configuration that is included into the message. If there's no new configuration versions and config body isn't included into message, then value of this field will equal to the one provided by endpoint.
- `status` (required) a human-readable string explaining the cause of an error (if any). In case that request was sucessful, it is `"ok"`.
- `config` (optional) - configuration body of an arbitrary JSON type.


Example:
```json
{
  "id": 42,
  "configVersion": 2,
  "status": "ok",
  "config": {
    "key": "value",
    "array" : [
        "value2"
    ]
  }
}
```

Example for the case when there's no new configuration version for endpoint:
```json
{
  "id": 42,
  "configVersion": 1,
  "status": "ok"
}
``` 

##### Configuration push
Client can listen for incoming endpoint configuration updates at the following resource path:
```
<endpoint_token>/push/json
```


The payload is a JSON record with the following fields:
- `id` (required) id of the message. Should be either string or number. Used to match server response to the request.
- `configVersion` (required) - version of configuration that is included into the message.
- `config` (required) - configuration body of an arbitrary JSON type.

Example:
```json
{
  "id": 42,
  "configVersion": 2,
  "config": [
    { "key": "value" },
    15,
    [ "an", "array", 13 ]
  ]
}
```

A delivery confirmation response must be sent to the next topic:
```
 <endpoint_token>/push/json/status
```
The confirmation message is a JSON record with the following fields:
- `id` (required) a copy of the `id` field from the corresponding request.
- `status` (required) a human-readable string explaining the cause of an error (if any). In case processing was sucessful, it is `"ok"`.
The destination topic is 

Example:
```json
{
  "id": 42,
  "status": "ok"
}
```