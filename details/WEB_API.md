# HTTP API

- **Base URL**: `http://<host>:<port>` (port from config file)
- **Content-Type**: `application/json`

## Responses

All business endpoints (excluding the health check) return responses in the following unified format.

### Success Response

```json
{"code": "", "data": <response_data>, "timestamp": 1741407495123}
```

| Field       | Type   | Description                                          |
|-------------|--------|------------------------------------------------------|
| `code`      | string | Error code. Empty string `""` indicates success        |
| `data`      | any    | Response payload (type depends on endpoint)            |
| `timestamp` | int    | Millisecond UTC timestamp                             |

### Error Response

```json
{"code": '<project>::<module>::<code>', "message": '<error description>', "timestamp": 1741407495123}
```

| Field       | Type   | Description                                          |
|-------------|--------|------------------------------------------------------|
| `code`      | string | Error code in format `project::module::code`           |
| `message`   | string | Human-readable error description                       |
| `timestamp` | int    | Millisecond UTC timestamp                             |

## Endpoints

### GET /hello

Health check. Returns `hello` (text/plain).

### <METHOD> /<resource>[/{id}]

<Description of what this endpoint does>.

**Path Parameters** (if applicable):
| Name | Type | Description |
|------|------|-------------|
| `id` | int  | Resource ID |

**Query Parameters** (if applicable):
| Name         | Type | Required | Default | Description                  |
|--------------|------|----------|---------|------------------------------|
| `field_type` | str  | No       | simple  | simple or full               |
| `page`       | int  | No       | 1       | Page number (from 1)         |
| `page_size`  | int  | No       | -       | Items per page (unlimited)   |

**Request Body** (if applicable):
```json
{"<field>": '<type>', ...}
```

**Success Example** (200 / 201 depending on operation):
```json
{
  "code": "",
  "data": { ... },
  "timestamp": 1741407495123
}
```

**Error Example**:
```json
{
  "code": 'project::module::code',
  "message": 'error description',
  "timestamp": 1741407495123
}
```
