# Sillage MCP Public Interface

This document describes how to use Sillage's local MCP server. It intentionally documents only public usage, tool inputs, public outputs, and privacy boundaries.

## Connection

Start MCP Server from Sillage settings. The app shows one or more endpoint URLs and an auth token.

```text
http://127.0.0.1:<port>/mcp
Authorization: Bearer <token>
```

If LAN access is enabled, use one of the LAN URLs shown by Sillage. If connecting from a desktop MCP client to an Android device during development, the user may need:

```text
adb reverse tcp:<port> tcp:<port>
```

MCP Server is a Sillage Plus feature. If access is not available, the app may show the subscription screen or the HTTP server may reject access.

## Results And Errors

Successful tool calls return a short text summary and machine-readable `structuredContent.data`.

Prefer:

```text
structuredContent.data
```

Common error categories:

- `invalid_params`: bad field, unsupported record type, invalid date, invalid time range, unknown airport/station/brand, ambiguous brand
- `resource_not_found`: record or candidate token not found
- `internal_error`: local service failure

## Capabilities

The app controls which tool groups are available.

| Capability | Tools |
| --- | --- |
| `read_records` | `search_records`, `get_record`, `create_backup_download` |
| `write_records` | create/edit tools, hotel candidate tools, brand name lookup, `delete_record` |

`delete_record` also requires deletion to be enabled in the Sillage MCP settings.

## Shared Field Rules

Tool inputs use camelCase.

Record types:

```text
flight | train | hotel | track
```

Flights, trains, and tracks use Unix millisecond timestamps. Hotels use date-only strings:

```text
YYYY-MM-DD
```

For create and edit tools:

- omit unchanged fields
- use `null` only to clear nullable fields
- keep `arrivalTime` later than `departureTime`
- keep `checkOutDate` later than `checkInDate`

## Tools

### `search_records`

Search redacted travel records.

Input:

```json
{
  "query": "Tokyo",
  "recordTypes": ["flight", "train", "hotel", "track"],
  "startTime": 1780000000000,
  "endTime": 1780172800000,
  "limit": 20
}
```

Notes:

- `recordTypes` is optional; by default all supported record types are searched.
- `limit` is capped by Sillage.
- Results are sorted by record time descending.
- Returned records do not include non-public lookup or map details.

### `get_record`

Read one redacted record by type and id.

Input:

```json
{
  "recordType": "hotel",
  "id": "hotel_xxx"
}
```

Example returned record:

```json
{
  "record": {
    "type": "hotel",
    "id": "hotel_xxx",
    "name": "Park Hyatt Tokyo",
    "description": null,
    "checkInDate": "2026-06-10",
    "checkOutDate": "2026-06-12",
    "hotelName": "Park Hyatt Tokyo",
    "country": "JP",
    "roomNumber": "1208",
    "roomType": "King",
    "confirmationNumber": "ABC123",
    "price": 1200
  }
}
```

### `create_flight`

Create a flight using airport codes already supported by Sillage. Do not pass airport coordinates.

Input:

```json
{
  "flightNumber": "CA1234",
  "airline": "CCA",
  "departureAirportCode": "PEK",
  "arrivalAirportCode": "SHA",
  "departureTime": 1780000000000,
  "arrivalTime": 1780007200000,
  "aircraftManufacturer": "Airbus",
  "aircraftType": "A320",
  "cabinClass": "Economy",
  "seatNumber": "12A",
  "bookingReference": "ABC123",
  "price": 800,
  "description": "Beijing to Shanghai"
}
```

Forbidden input fields:

- `departureLatitude`
- `departureLongitude`
- `arrivalLatitude`
- `arrivalLongitude`

### `edit_flight`

Patch a flight by id. Omitted fields are preserved.

Input:

```json
{
  "id": "flight_xxx",
  "seatNumber": "12C",
  "bookingReference": null
}
```

### `create_train`

Create a train record. Station names or station codes are accepted. Coordinates are accepted only when explicitly provided by the user, and they are not returned by read tools.

Input:

```json
{
  "trainNumber": "G11",
  "trainType": "G",
  "departureStation": "Beijing South",
  "arrivalStation": "Shanghai Hongqiao",
  "departureTime": 1780000000000,
  "arrivalTime": 1780018000000,
  "carriage": "02",
  "seatNumber": "02A",
  "seatType": "Second Class",
  "ticketNumber": "E123456",
  "passengerName": "Passenger",
  "railwaySystem": "CR",
  "price": 553,
  "description": "Beijing to Shanghai"
}
```

Optional user-provided coordinate pairs:

- `departureLatitude` and `departureLongitude`
- `arrivalLatitude` and `arrivalLongitude`

### `edit_train`

Patch a train by id. Omitted fields are preserved.

Input:

```json
{
  "id": "train_xxx",
  "seatNumber": "03F",
  "ticketNumber": null
}
```

### `list_hotel_brand_names`

List English hotel brand names accepted by hotel create/edit tools.

Input:

```json
{
  "query": "Hyatt",
  "limit": 50
}
```

Returned data:

```json
{
  "brands": [
    { "brandName": "Park Hyatt" },
    { "brandName": "Hyatt Regency" }
  ]
}
```

Pass the returned `brandName` exactly.

### `search_hotel_candidates`

Search hotel candidates. Each candidate returns only a short-lived token and one localized hotel name.

Input:

```json
{
  "query": "Park Hyatt",
  "locale": "en-US",
  "limit": 5
}
```

Returned data:

```json
{
  "candidates": [
    {
      "hotelCandidateToken": "hotel_candidate_xxx",
      "hotelName": "Park Hyatt Tokyo"
    }
  ]
}
```

Use only the returned candidate token and hotel name. Do not ask for or infer hidden candidate details.

### `create_hotel_manual`

Create a hotel stay from fields explicitly provided by the user.

Input:

```json
{
  "hotelName": "Park Hyatt Tokyo",
  "brandName": "Park Hyatt",
  "country": "JP",
  "roomNumber": "1208",
  "roomType": "King",
  "checkInDate": "2026-06-10",
  "checkOutDate": "2026-06-12",
  "latitude": 35.6852,
  "longitude": 139.6917,
  "confirmationNumber": "ABC123",
  "price": 1200,
  "description": "Tokyo stay"
}
```

Rules:

- `brandName`, if provided, must uniquely match an accepted brand name.
- coordinates are optional and must come from the user
- `latitude` and `longitude` must be provided together

### `create_hotel_from_candidate`

Create a hotel stay from a candidate token returned by `search_hotel_candidates`.

Input:

```json
{
  "hotelCandidateToken": "hotel_candidate_xxx",
  "checkInDate": "2026-06-10",
  "checkOutDate": "2026-06-12",
  "roomNumber": "1208",
  "price": 1200
}
```

Optional user-provided overrides:

```json
{
  "hotelCandidateToken": "hotel_candidate_xxx",
  "hotelName": "Park Hyatt Tokyo",
  "brandName": "Park Hyatt",
  "country": "JP",
  "latitude": 35.6852,
  "longitude": 139.6917,
  "checkInDate": "2026-06-10",
  "checkOutDate": "2026-06-12"
}
```

Use the candidate token as an opaque value.

### `edit_hotel`

Edit a hotel stay.

Input:

```json
{
  "id": "hotel_xxx",
  "hotelName": "Park Hyatt Tokyo",
  "brandName": "Park Hyatt",
  "country": "JP",
  "roomNumber": "1208",
  "roomType": "King",
  "checkInDate": "2026-06-10",
  "checkOutDate": "2026-06-12",
  "confirmationNumber": "ABC123",
  "price": 1200,
  "description": "Tokyo stay"
}
```

### `create_backup_download`

Create a Sillage backup download only when the user asks to export their data.

The returned download URL is short-lived and belongs to the same MCP server origin. Use the same bearer token when downloading.

### `delete_record`

Delete one record. Confirm with the user before calling this tool.

Input:

```json
{
  "recordType": "hotel",
  "id": "hotel_xxx"
}
```

Deletion must be enabled in Sillage MCP settings.

## Privacy Boundary

Only use data returned by MCP tools or explicitly provided by the user.

Do not request or expose hidden lookup details, generated map details, background processing data, credentials, email contents, raw SQL, database dumps, arbitrary local file paths, or any non-public app data.

Coordinates may appear in tool inputs only when the user explicitly supplies them. Some tools may use opaque candidate tokens, but those tokens should not be expanded or described.
