# API Versioning

## Summary

This document describes a strategy for versioning public APIs. The goal is to keep integrations stable while the platform keeps evolving.

The strategy combines two mechanisms:

- A stable major version in the URL path, such as `/v1`, for all REST APIs and webhooks.
- A date-based version header, such as `Api-Version: 2025-03-01`, for ongoing changes.

The path version is the stable foundation. It changes only in extreme cases, such as a major redesign of the data model. The header allows controlled evolution under that path. Clients send the header to pin a version. Clients that omit it get the latest stable release.

All non-major releases are backward compatible, so clients can adopt them safely. Major releases may include breaking changes. They carry a name suffix so they stand out.

Internally, the system always runs on the latest schema and logic. Transformers adapt requests and responses between versions. When business logic must differ by version, an explicit version check gates it.

## Problem

Once APIs are public, we want to change them quickly. Changes improve the platform and respond to client needs. For each change, we need to know whether it breaks existing integrations. We also need to tell clients, so they don't move to a version that is incompatible with their code.

## Solution

Versioning is the industry norm for this. Each class of release gets its own versioning scheme. Developers then build an intuition for whether a release is safe to adopt. They don't need to analyze every change or decode version numbers.

### Principles

1. **API contracts are forever.**
   Once a version is published, clients pinned to it keep working as the API evolves. The only exception is a new legal, security, or correctness requirement that makes compatibility impossible. In those rare cases, the break stays narrow. Errors are clear and consistent. Clients get advance notice and migration guidance.
2. **Contracts are separate from business logic.**
   API contracts are stable interfaces. They are not tightly coupled to the internal implementation. Core logic always evolves on the latest schema. A compatibility layer handles contract changes such as renamed fields, added or removed fields, and changed data types. If a business rule must differ across versions, the main logic gates it explicitly.
3. **Clients control their version.**
   Clients are never silently upgraded. Each client can pin a version. New features and new required inputs apply only to newer versions. Changelogs, versioned docs, and preview tools let clients upgrade at their own pace.

## Proposal

- **Use one global version (`/v1`) across all REST APIs and webhooks.** Non-HTTP integrations, such as batch file transfers, are out of scope.
  - *Why:* A typical integration calls many endpoints. One version across the platform simplifies change management.
- **Accept an `Api-Version` header. Default to the latest stable version when it is absent.**
  - *Why:* Clients control which version they use and when they move. Minor version bumps need no other change, such as a new URI.
- **Name versions by release date.**
  - *Why:* A date says more than a bare number.
- **Every non-major release MUST be safe.** A client can change the `Api-Version` header and everything still works.
  - *Why:* We ship capabilities quickly while clients stay safe from breakage.
- **Give major releases a name suffix,** such as `2025-03-01.basil`.
  - *Why:* Names are easier to tell apart than letters or numbers. They also build intuition that this release is different.
- **Publish a changelog for every client-facing change.**
  - *Why:* It is industry best practice. Clients get one place to see what's new and decide whether to act.

### Benefits

- Builds developer trust, because integrations don't break unexpectedly.
- Scales with rapid product change while still serving older clients.
- Gives a clear process for regulatory changes that force unavoidable breaks.
- Lowers operational risk through predictable behavior, observability, and clear communication.

## Implications

The platform team needs to:

- Batch version bumps into a fixed cadence, such as monthly. Regular deployments that don't touch API contracts continue as usual.
- Track which version each change belongs to.
- Add an unstable path, such as `/v0` or `/unstable`, for private betas and experimental features.
- Add automated checks that classify each change as breaking or non-breaking.
- Check the version header on each request. Default to the latest stable version. Return a response header with the version used to serve the request.
- Support serving many versions in parallel.
- Generate changelog entries automatically and tie them to versions.

### Internal Version Handling

The system always runs on the latest schema and business logic. Each request resolves a target version from the `Api-Version` header or the default rules. A chain of transformers then honors that version.

```
request (v: 2024-06)                                   response (v: 2024-06)
     |                                                        ^
     v                                                        |
[inbound: 2024-06 -> 2024-09 -> 2025-01 -> latest]   [outbound: latest -> 2025-01 -> 2024-09 -> 2024-06]
     |                                                        ^
     v                                                        |
     +-------------> business logic (latest schema) ----------+
```

- **Inbound transformers (old to latest).** Request parameters from an older version move forward step by step until they match the latest schema. Each transformer represents one published change. They apply in chronological order.
- **Business logic.** Core services run only on the latest schema. They know nothing about legacy shapes. Any version-specific behavior is explicitly gated on the resolved version.
- **Outbound transformers (latest to old).** Responses move backward from the latest schema to the requested version. Transformers apply in reverse order until they reach the target.

Results:

- All data at rest is stored in the latest form.
- Core logic stays clean and current.
- A well-defined transformer chain provides compatibility.

### Side Effect Handling

Some changes alter behavior, not just shape. These side effects are explicitly marked and reviewed. The main business logic gates them by checking the resolved version. It never infers the version from whether a transformer ran. Each side effect needs approval, so it is justified, documented, and testable.

### Other Considerations

- **Governance:** Releases follow a predictable process that is well communicated.
- **Developer experience:** Version-aware docs and SDKs make adoption easy.
- **Observability:** Metrics and logs track version usage.
- **Communication:** Changelogs and announcements keep developers informed.
- **Testing:** Fixtures and regression tests cover every supported version.

## Prior Art

### Stripe

- URL paths are stable (`/v1`).
- The version is sent in a `Stripe-Version` header.
  ```sh
  curl -G https://api.stripe.com/v1/charges \
    -H "Authorization: Bearer $API_KEY" \
    -H "Stripe-Version: 2024-09-30"
  ```
- Versions are named by monthly release date.
- Monthly releases are backward compatible.
- Major releases, roughly twice a year, add a name suffix, such as `2025-03-01.basil`. These may contain breaking changes.
- Changelog: <https://docs.stripe.com/changelog>

### Shopify

- The version is part of the URL, such as `/admin/api/2025-07/graphql.json`.
- There are three version types: stable, release candidate, and unstable.
- Unstable versions use `unstable` in place of the date, such as `/admin/api/unstable/graphql.json`. These serve private betas.
- Responses include an `X-Shopify-API-Version` header with the version that served the request. A mismatch with the requested version means the client is out of date and fell back to the default.
- Changelog: <https://shopify.dev/changelog>

### Salesforce

- One global version number across APIs, bumped with each major release, about three times a year.
- Each major release is the point where API families can ship breaking changes.
- Each version is supported for at least 3 years. Customers get at least 1 year notice before a version is retired.
- Requests to a retired version return `410 Gone`.
- Clients declare the version explicitly, for example:
  ```xml
  <apiVersion>58.0</apiVersion>
  ```

## References

- <https://docs.stripe.com/api/versioning>
- <https://shopify.dev/docs/api/usage/versioning>
