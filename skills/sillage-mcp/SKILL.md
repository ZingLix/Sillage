---
name: sillage-mcp
description: Use when managing a user's Sillage travel records through Sillage's local MCP server, including searching, reading, creating, editing, backing up, or deleting flight, train, hotel, and track records while following the public MCP interface and privacy-safe data rules.
---

# Sillage MCP

Use Sillage's local MCP server to manage the user's own travel records. Treat Sillage as a local-first travel journal controlled by the user.

For the complete public interface shape, read `references/mcp-api.md` in this skill folder.

## Workflow

1. Ask the user to start MCP Server in Sillage and provide the endpoint URL and token shown in the app.
2. Connect to the Streamable HTTP endpoint shown by Sillage.
3. Use `Authorization: Bearer <token>` for every request.
4. Prefer `structuredContent.data` for results; use text content only as a short summary.
5. Omit unchanged fields when editing. Use `null` only when the user explicitly wants to clear a nullable field.
6. Confirm before destructive actions. `delete_record` only works when deletion is enabled in Sillage MCP settings.

## Data Rules

Allowed to use and return:

- record ids, record type, name, description
- flight number, airline, airport codes, aircraft, cabin, seat, booking reference
- train number/type, station names/codes, carriage, seat, ticket number
- hotel name, date-only stay range, country, room details, confirmation number
- user-provided coordinates when the user explicitly supplies them
- distance, duration, price, and notes when already part of the user's record

Never request or expose:

- non-public app data returned from lookup, matching, map, or collection systems
- hidden ids, raw lookup results, generated geometry, or background processing data
- OAuth tokens, remote-storage credentials, email body content, raw SQL, database dumps, or arbitrary local file paths

## Common Tasks

- Search records with `search_records`.
- Read one record with `get_record`.
- Create or patch flights with airport codes only.
- Create or patch trains with station names/codes; only include coordinates if the user explicitly provided them.
- Create hotels manually from user-provided fields, or search candidates and create from a short-lived candidate token.
- List acceptable hotel brand names with `list_hotel_brand_names`; pass returned English `brandName` values exactly.
- Create an app backup download only when the user asks for an export.

## Notes

MCP Server is a Sillage Plus feature. If the server cannot start or returns payment-required access errors, ask the user to unlock Sillage Plus or disable MCP-related automation.
