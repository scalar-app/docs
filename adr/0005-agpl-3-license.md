# 0005. AGPL-3.0 license

Date: 2026-08-18
Status: Accepted

## Context

Scalar is developed in public and intended to be self-hostable. A hosted service may exist later. The project wants improvements made to hosted versions to flow back to the community, and wants to keep the option of offering the same code as a service.

## Decision

Every repository is licensed under AGPL-3.0-only. Contributions are accepted inbound under the same license (inbound equals outbound), without a CLA.

## Consequences

- Anyone can run, read and change Scalar, including as a service, provided they publish their changes.
- Some organizations avoid AGPL dependencies; `@scalar/sdk` and `@scalar/ui` are also AGPL for now, which limits third party adoption of the client packages. This can be revisited with a separate record if a permissive license for the SDK becomes important.
- Every repository states the license in `LICENSE`, `package.json` and its README.

## Alternatives considered

- MIT or Apache-2.0: maximum adoption, no obligation to share hosted improvements.
- Business source or source available licenses: not open source; rejected.
