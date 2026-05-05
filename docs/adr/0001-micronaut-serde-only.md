# ADR-0001 — Use Micronaut Serde for all JSON serialization (no direct Jackson use)

| | |
|--|--|
| **Status** | Accepted |
| **Date** | 2026-05-05 |
| **Decider(s)** | Rui (Tech Lead), Tomás (Security Champion) |
| **Owner ticket** | ARCH-08 (cross-repo); ARCH-08-followup-revolut (this MR) |
| **Scope** | `backend/revolut-webhook` — this repo. |
| **Sibling ADR** | `stripe-webhook/docs/adr/0008-micronaut-serde-only.md` (merged 2026-05-05). |

## Context

`revolut-webhook` is currently in bootstrap state (commits `de1a478` → `9de5b8d`) — only the Lambda runtime skeleton + a stub handler. No business logic has shipped yet; the `build.gradle` declares only `io.micronaut.serde:micronaut-serde-jackson` (the Micronaut Serde implementation), with **no direct `com.fasterxml.jackson.core:jackson-databind` dependency**.

This MR locks the same architectural rule the sibling `stripe-webhook` repo adopted in ADR-0008: **production code in `src/main/` MUST NOT directly import `com.fasterxml.jackson.*`**. Micronaut Serde provides typed JSON binding via `@Serdeable` records.

The discipline is being established **before** any business logic lands, so the constraint is prevention-only: there is nothing to migrate. The next contributor to ship a Revolut webhook handler will write `@Serdeable` records from day one, never `JsonNode`.

## Decision

**`revolut-webhook` shall use Micronaut Serde for all JSON serialization and deserialization in production code. Direct `com.fasterxml.jackson.core:jackson-databind` imports in `src/main/` are forbidden.**

The rationale matches `stripe-webhook` ADR-0008 verbatim:

1. **Deserialization-gadget surface (default-deny).** Jackson's polymorphic-deserialization features (`@JsonTypeInfo(use = Id.CLASS)`, `enableDefaultTyping`) are the root of a long history of CVEs. Micronaut Serde's `@Serdeable` processor generates static, build-time deserializers per record — the polymorphic API doesn't exist in the surface. Removing the API is the structural fix.
2. **Typed events at the trust boundary.** Webhook bodies are untrusted input. Typed `@Serdeable` records bind the wire shape explicitly; field renames at Revolut's end surface as parse errors at the Lambda boundary, not as runtime `MissingNode.path(...)` traversals at handler level.
3. **Single JSON layer.** If we ever add `revolut-business-java` (Revolut's official SDK), it stays inside an adapter package; the layer's wire types don't surface past the adapter. Locking on Micronaut Serde now prevents three JSON libraries in one Lambda.

### Concrete changes (this MR)

1. **ADR (this file).**
2. **CI static check** in `.github/workflows/build.yml` — a new step that runs before `./gradlew build`, fail-fast on match. Pattern is unanchored `com.fasterxml.jackson` (catches both `import` lines AND fully-qualified-name uses):
   ```yaml
   - name: ADR-0001 — no direct Jackson use in production code
     run: |
       OFFENDERS=$(find src/main -name "*.java" -exec grep -l "com\.fasterxml\.jackson" {} + || true)
       if [ -n "$OFFENDERS" ]; then
         echo "::error::Direct Jackson references found in production code (forbidden — see docs/adr/0001-micronaut-serde-only.md):"
         echo "$OFFENDERS"
         exit 1
       fi
       echo "✓ No direct Jackson references in production code."
   ```
   The check lives **before** the gradle build so it fails inside seconds rather than waiting for the full native-image compile.

### What this MR does NOT do

- No code change. Bootstrap state has no `JsonNode` to migrate.
- No `build.gradle` change. The repo never declared a direct `jackson-databind` dependency, so there's nothing to remove.
- No future-typed-event-record decisions. When the first Revolut webhook handler ships, it will:
   - Define a typed sealed `RevolutEvent` hierarchy (mirroring stripe-webhook's `StripeEvent` shape).
   - Bind the wire envelope via an `@Serdeable record RevolutEventEnvelope(...)`.
   - Pattern-match in the dispatcher with a switch expression — no `default` branch (sealed-completeness).
   That work lives under separate tickets (PAY-0X for Revolut webhook coverage).

## Consequences

### Positive

- The CI gate is in place from day one. The first contributor to write a Revolut webhook handler cannot accidentally import `JsonNode` or `ObjectMapper` — the build fails before `./gradlew build` runs.
- Cross-repo consistency. `stripe-webhook` and `revolut-webhook` share the same architectural rule, the same CI grep pattern, the same ADR text shape. Cognitive overhead drops.
- Prevention is cheap. ~30 lines total (this ADR + the CI step). No code review overhead at the time of first business-logic MR.

### Negative / cost

- **None at present.** When the first Revolut handler ships, the contributor must write Serde-bound records. That's the same shape required by every other Sobrado backend Lambda (`payment-lambda`, `data-handler`, `stripe-webhook`, `pii-ingestion`), so there's no new learning cost for the team.

## Cross-codebase consistency note

`stripe-webhook` ADR-0008 is the canonical version of this rule. This ADR's existence is a localised lock — having the same rule documented + enforced in each repo where the rule applies, rather than relying on a single document elsewhere. Future audit ("does revolut-webhook follow the no-Jackson-in-production rule?") finds the answer inside the repo.

If a future cross-repo decision changes the rule (e.g., we discover a Micronaut Serde limitation that forces direct Jackson use somewhere), updates land in BOTH ADRs together, with the canonical reasoning in one ADR and a "follows the canonical rule per `<canonical-ADR-link>`" pointer in the other.

## Verification

- `find src/main -name "*.java" -exec grep -l "com.fasterxml.jackson" {} +` returns empty in CI (verified at MR-time on `9de5b8d`).
- `./gradlew dependencies | grep jackson-databind` shows `jackson-databind` only as a transitive of `io.micronaut.serde:micronaut-serde-jackson`, never at depth 1.

## Follow-ups

- **No specific follow-ups for revolut-webhook.** The discipline is now locked; subsequent Revolut webhook coverage tickets pick up the rule automatically.
- **Cross-repo alignment** with `pii-ingestion`: PR #8 of stripe-webhook also added a structural CI grep guard on `StripeEvent.Ignored.unrecognisedType()`. If `pii-ingestion`'s sealed `DispatchResult.UnknownField` ever grows a similar debug-only string field, the same allow-list-grep pattern applies — track via the existing typed-event-discipline practice rather than a new ADR.
