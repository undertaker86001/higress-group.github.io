---
title: Cluster Rate Limiting Based on Key  
keywords: [higress, rate-limit]  
description: Configuration reference for the Key-based cluster rate limiting plugin

---

## Function Description

The `cluster-key-rate-limit` plugin implements **cluster-level rate limiting** based on Redis, suitable for scenarios
requiring **globally consistent rate limiting across multiple Higress Gateway instances**.

It supports two rate limiting modes:

- **Rule-Level Global Rate Limiting**: Applies a unified rate limit threshold to custom rule groups based on identical `rule_name` and `global_threshold` configurations.
- **Key-Level Dynamic Rate Limiting**: Groups and limits requests by dynamic keys extracted from requests, such as URL parameters, request headers, client IPs, consumer names, or cookie fields.

## Operational Attributes

- **Plugin execution phase**: `Default phase`
- **Plugin execution priority**: `20`

## Configuration Instructions

| Configuration Item       | Type          | Required                                  | Default Value       | Description                                                                |
|--------------------------|---------------|-------------------------------------------|---------------------|----------------------------------------------------------------------------|
| rule_name                | string        | Yes                                       | -                   | Name of the rate limiting rule. Used to construct the Redis key in the format: `rule_name:rate_limit_type:key_name:key_value`. |
| global_threshold         | Object        | No (choose either `global_threshold` or `rule_items`) | -                 | Apply rate limiting to the entire custom rule group.|
| rule_items               | array of object | No (choose either `global_threshold` or `rule_items`) | -               | The rate limiting rule items array is **matched in order**. When the first qualifying rule item is hit, rate limiting triggers and **subsequent rules are not executed**. |
| show_limit_quota_header  | bool          | No                                        | false             | Whether to display `X-RateLimit-Limit` (total allowed requests) and `X-RateLimit-Remaining` (remaining allowed requests) in the response header. |
| rejected_code            | int           | No                                        | 429               | HTTP status code returned when a request is rate-limited.                  |
| rejected_msg             | string        | No                                        | Too many requests | Response body returned when a request is rate-limited.                      |
| redis                    | object        | Yes                                       | -                   | Configuration for Redis.                                                   |

### Configuration Fields for `global_threshold`

| Configuration Item       | Type | Required                                 | Default Value | Description                          |  
|--------------------------|------|------------------------------------------|---------------|--------------------------------------|  
| query_per_second         | int  | No (choose one of `query_per_second`, `query_per_minute`, `query_per_hour`, `query_per_day`) | -           | Allowed requests per second.         |  
| query_per_minute         | int  | No (choose one of `query_per_second`, `query_per_minute`, `query_per_hour`, `query_per_day`) | -           | Allowed requests per minute.         |  
| query_per_hour           | int  | No (choose one of `query_per_second`, `query_per_minute`, `query_per_hour`, `query_per_day`) | -           | Allowed requests per hour.           |  
| query_per_day            | int  | No (choose one of `query_per_second`, `query_per_minute`, `query_per_hour`, `query_per_day`) | -           | Allowed requests per day.            |  

### Configuration Fields for `rule_items`

| Configuration Item            | Type          | Required                          | Default Value | Description                                                                 |  
|-------------------------------|---------------|-----------------------------------|---------------|-----------------------------------------------------------------------------|  
| limit_by_header               | string        | No (choose one of `limit_by_*` fields) | -           | Configures the HTTP request header name to extract the rate limiting key.   |  
| limit_by_param                | string        | No (choose one of `limit_by_*` fields) | -           | Configures the URL parameter name to extract the rate limiting key.        |  
| limit_by_consumer             | string        | No (choose one of `limit_by_*` fields) | -           | Rate limits based on the consumer name (no need to add a specific value).   |  
| limit_by_cookie               | string        | No (choose one of `limit_by_*` fields) | -           | Configures the Cookie key name to extract the rate limiting key.           |  
| limit_by_per_header           | string        | No (choose one of `limit_by_*` fields) | -           | Matches specific HTTP headers by rule and calculates rate limits for each header. Supports regular expressions (starting with `regexp:`) or `*` for the `limit_keys` configuration. |  
| limit_by_per_param            | string        | No (choose one of `limit_by_*` fields) | -           | Matches specific URL parameters by rule and calculates rate limits for each parameter. Supports regular expressions (starting with `regexp:`) or `*` for the `limit_keys` configuration. |  
| limit_by_per_consumer         | string        | No (choose one of `limit_by_*` fields) | -           | Matches specific consumers by rule and calculates rate limits for each consumer. Supports regular expressions (starting with `regexp:`) or `*` for the `limit_keys` configuration (no need to add a specific value for the consumer name). |  
| limit_by_per_cookie           | string        | No (choose one of `limit_by_*` fields) | -           | Matches specific Cookies by rule and calculates rate limits for each Cookie value. Supports regular expressions (starting with `regexp:`) or `*` for the `limit_keys` configuration. |  
| limit_by_per_ip               | string        | No (choose one of `limit_by_*` fields) | -           | Matches specific IPs by rule and calculates rate limits for each IP. The IP can be extracted from a request header (formatted as `from-header-<header_name>`, e.g., `from-header-x-forwarded-for`) or directly from the peer socket IP (configured as `from-remote-addr`). |  
| limit_keys                    | array of object | Yes                               | -           | Configures the rate limits for matched key values.                          |  

### Configuration Fields for `limit_keys`

| Configuration Item       | Type   | Required                                 | Default Value | Description                                                                 |  
|--------------------------|--------|------------------------------------------|---------------|-----------------------------------------------------------------------------|  
| key                      | string | Yes                                      | -             | The matched key value. For `limit_by_per_header`, `limit_by_per_param`, `limit_by_per_consumer`, and `limit_by_per_cookie` types, supports regular expressions (prefixed with `regexp:`) or `*` (wildcard for all). Example regular expression: `regexp:^d.*` (matches all strings starting with `d`). For `limit_by_per_ip`, supports IP addresses or CIDR blocks. |  
| query_per_second         | int    | No (choose one of `query_per_second`, `query_per_minute`, `query_per_hour`, `query_per_day`) | -           | Allowed requests per second.                                                |  
| query_per_minute         | int    | No (choose one of `query_per_second`, `query_per_minute`, `query_per_hour`, `query_per_day`) | -           | Allowed requests per minute.                                                |  
| query_per_hour           | int    | No (choose one of `query_per_second`, `query_per_minute`, `query_per_hour`, `query_per_day`) | -           | Allowed requests per hour.                                                  |  
| query_per_day            | int    | No (choose one of `query_per_second`, `query_per_minute`, `query_per_hour`, `query_per_day`) | -           | Allowed requests per day.                                                   |  

### Configuration Fields for `redis`

| Configuration Item   | Type   | Required | Default Value                                                     | Description                                                                 |  
|----------------------|--------|----------|-------------------------------------------------------------------|-----------------------------------------------------------------------------|  
| service_name         | string | Yes      | -                                                                 | The fully qualified domain name (FQDN) of the Redis service, including the service type (e.g., `my-redis.dns`, `redis.my-ns.svc.cluster.local`). |  
| service_port         | int    | No       | 80 (for static services), 6379 for other services                  | The port of the Redis service.                                              |  
| username             | string | No       | -                                                                 | Redis username for authentication.                                          |  
| password             | string | No       | -                                                                 | Redis password for authentication.                                          |  
| timeout              | int    | No       | 1000 (milliseconds)                                               | Redis connection timeout in milliseconds.                                  |  
| database             | int    | No       | 0                                                                 | The ID of the Redis database to use (e.g., configuring `1` corresponds to `SELECT 1`). |  

## Configuration Examples

### Global Rate Limiting for Custom Rule Group

```yaml  
rule_name: routeA-global-limit-rule
global_threshold:
  query_per_minute: 1000 # Global limit: 1000 requests/minute for the custom rule group
redis:
  service_name: redis.static
show_limit_quota_header: true
```

### Rate Limiting by Request Parameter `apikey`

```yaml  
rule_name: routeA-request-param-limit-rule
rule_items:
  - limit_by_param: apikey
    limit_keys:
      - key: 9a342114-ba8a-11ec-b1bf-00163e1250b5 # Fixed value1: 10 requests/minute
        query_per_minute: 10
      - key: a6a6d7f2-ba8a-11ec-bec2-00163e1250b5 # Fixed value2: 100 requests/hour
        query_per_hour: 100
  - limit_by_per_param: apikey
    limit_keys:
      - key: "regexp:^a.*" # Regex match a-starting apikey: 10 requests/second per key
        query_per_second: 10
      - key: "regexp:^b.*" # Regex match b-starting apikey: 100 requests/minute per key
        query_per_minute: 100
      - key: "*" # Fallback rule: all apikeys, 1000 requests/hour per key
        query_per_hour: 1000
redis:
  service_name: redis.static
show_limit_quota_header: true
```

### Rate Limiting by Request Header `x-ca-key`

```yaml  
rule_name: routeA-request-header-limit-rule
rule_items:
  - limit_by_header: x-ca-key
    limit_keys:
      - key: 102234 # Fixed value1: 10 requests/minute
        query_per_minute: 10
      - key: 308239 # Fixed value2: 10 requests/hour
        query_per_hour: 10
  - limit_by_per_header: x-ca-key
    limit_keys:
      - key: "regexp:^a.*" # Regex match a-starting x-ca-key: 10 requests/second per key
        query_per_second: 10
      - key: "regexp:^b.*" # Regex match b-starting x-ca-key: 100 requests/minute per key
        query_per_minute: 100
      - key: "*" # Fallback rule: all x-ca-keys, 1000 requests/hour per key
        query_per_hour: 1000
redis:
  service_name: redis.static
show_limit_quota_header: true
```

### Rate Limiting by Client IP Extracted from `x-forwarded-for` Header

```yaml  
rule_name: routeA-client-ip-limit-rule
rule_items:
  - limit_by_per_ip: from-header-x-forwarded-for # IP rate limiting via x-forwarded-for header
    limit_keys:
      - key: 1.1.1.1 # Exact IP: 1.1.1.1, 10 requests/day
        query_per_day: 10
      - key: 1.1.1.0/24 # IP range 1.1.1.0/24: 100 requests/day per IP
        query_per_day: 100
      - key: 0.0.0.0/0 # Fallback rule: all IPs, 1000 requests/day per IP
        query_per_day: 1000
redis:
  service_name: redis.static
show_limit_quota_header: true
```

### Rate Limiting by Consumer

```yaml  
rule_name: routeA-consumer-limit-rule
rule_items:
  - limit_by_consumer: ''
    limit_keys:
      - key: consumer1 # Consumer1: 10 requests/second
        query_per_second: 10
      - key: consumer2 # Consumer2: 100 requests/hour
        query_per_hour: 100
  - limit_by_per_consumer: ''
    limit_keys:
      - key: "regexp:^a.*" # Regex match a-starting consumer: 10 requests/second per key
        query_per_second: 10
      - key: "regexp:^b.*" # Regex match b-starting consumer: 100 requests/minute per key
        query_per_minute: 100
      - key: "*" # Fallback rule: all consumers, 1000 requests/hour per key
        query_per_hour: 1000
redis:
  service_name: redis.static
show_limit_quota_header: true
```

### Rate Limiting by Cookie Value

```yaml  
rule_name: routeA-cookie-limit-rule
rule_items:
  - limit_by_cookie: key1 # Rate limit by Cookie key "key1"
    limit_keys:
      - key: value1 # Fixed value1: 10 requests/minute
        query_per_minute: 10
      - key: value2 # Fixed value2: 100 requests/hour
        query_per_hour: 100
  - limit_by_per_cookie: key1 # Rate limit by each value of Cookie key "key1"
    limit_keys:
      - key: "regexp:^a.*" # Regex match a-starting value: 10 requests/second per key
        query_per_second: 10
      - key: "regexp:^b.*" # Regex match b-starting value: 100 requests/minute per key
        query_per_minute: 100
      - key: "*" # Fallback rule: all values, 1000 requests/hour per key
        query_per_hour: 1000
rejected_code: 200 # HTTP status code for rate-limited requests
rejected_msg: '{"code":-1,"msg":"Too many requests"}' # Response body for rate-limited requests
redis:
  service_name: redis.static
show_limit_quota_header: true
```