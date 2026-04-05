# HTTP API

- **Base URL**: `http://<host>:<port>` (port from config file)
- **Content-Type**: `application/json`

## Responses

All endpoints return responses in the following unified format.

### Success Response

```json
{"code": "", "data": <response_data>, "timestamp": "2026-03-08T05:38:15.123"}
```

| Field       | Type   | Description                                          |
|-------------|--------|------------------------------------------------------|
| `code`      | string | Error code. Empty string `""` indicates success        |
| `data`      | any    | Response payload (type depends on endpoint)            |
| `timestamp` | string | UTC time in format `%Y-%m-%dT%H:%M:%S.%f` (milliseconds, no trailing `Z`) |

### Error Response

```json
{"code": "<namespace>::<module>::<error_code>", "message": "<error description>", "timestamp": "2026-03-08T05:38:15.123"}
```

| Field       | Type   | Description                                          |
|-------------|--------|------------------------------------------------------|
| `code`      | string | Error code in format `project::module::code`           |
| `message`   | string | Human-readable error description                       |
| `timestamp` | string | UTC time in format `%Y-%m-%dT%H:%M:%S.%f` (milliseconds, no trailing `Z`) |

## Endpoints

### GET /hello

Health check. Returns `hello` (text/plain).

### POST /<resource>

<Description of what this endpoint does>.

**Request**:
```json
{"<field>": <type>, ...}
```

**Success Example**:
```json
{
  "code": "",
  "data": { ... },
  "timestamp": "2026-03-08T05:38:15.123"
}
```

**Error Example**:
```json
{
  "code": "project::module::error_code",
  "message": "error description",
  "timestamp": "2026-03-08T05:38:15.123"
}
```
