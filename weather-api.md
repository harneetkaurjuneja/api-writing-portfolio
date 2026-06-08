## Get Weather

Returns current weather data for a specified city.

**Endpoint:** `GET /v1/weather`

### Request Parameters

| Parameter | Type   | Required | Description                        |
|-----------|--------|----------|------------------------------------|
| city      | string | Yes      | Name of the city                   |
| units     | string | No       | Unit system: metric or imperial    |

### Example Request

```json
{
  "city": "Vancouver",
  "units": "metric"
}
```
### Response Fields

| Field       | Type   | Description                              |
|-------------|--------|------------------------------------------|
| city        | string | Name of the requested city               |
| temperature | string | Current temperature in Celsius           |
| units       | string | Unit system — metric or imperial         |
| conditions  | string | Description of current weather           |
| humidity    | string | Percentage of moisture in the air        |

### Error Codes

| Code | Description                                    |
|------|------------------------------------------------|
| 200  | Success — data returned correctly              |
| 400  | Bad request — check your parameters            |
| 404  | City not found — check spelling and try again  |
| 500  | Server error — try again later                 |

### Error Response Example
```json
{
  "error": "404",
  "message": "City not found. Check the spelling and try again."
}
```
