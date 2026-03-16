# Project Guidelines

## Overview
This project contains a Santiment API skill for AI agents. The skill teaches agents how to use the Santiment GraphQL API and Python `sanpy` library to fetch cryptocurrency market data.

## Behavioral Rules
- Authentication uses `SANTIMENT_API_KEY` environment variable — never hardcode API keys.
- GraphQL errors return HTTP 200 — always check for `"errors"` key in response body, not just status code.
- Each GraphQL **query** inside a request counts as a separate API call for rate limiting purposes.
- Relative dates like `"utc_now-7d"` are server-side — don't compute UTC offsets client-side when these suffice.
- The `Content-Type` header must be `application/json` with `{"query": "..."}` body format (not `application/graphql`).

## Skill Location
The skill is at `.claude/skills/santiment-api.md`. It is invoked by AI agents when they need to work with Santiment crypto data.
