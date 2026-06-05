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

### Response

Returns current temperature, humidity, and weather conditions for the specified city.
