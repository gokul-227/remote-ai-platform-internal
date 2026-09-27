# ADR-004 — Typed API contracts come from the backend's OpenAPI

Status: accepted, 2026-09-27 (Phase 03; applied from Phase 04 on).

**Context.** 332 lint warnings in the product UI are almost all `any`, mostly `useApi<any>` (88 call sites). Only 91 of
174 API operations declare a response model; the rest return untyped dicts, which also means no schema filters their
fields (a privacy risk for public endpoints).

**Decision.** The backend's Pydantic response models are the contract. Every endpoint touched from Phase 04 on gets a
`response_model`; public ones get a public-only schema. Once the launched journeys' endpoints are typed, generate TS
types from `openapi.json` and add a CI drift check. Hand-written frontend DTOs only as a bridge.

**Consequences.** `any` shrinks as endpoints are typed, not through blanket casts. No new lint/type suppressions.
