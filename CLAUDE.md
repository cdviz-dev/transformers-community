# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Community-driven transformers for cdviz-collector that convert various event sources (GitHub, ArgoCD, Kubewatch) into CDEvents format. Transformers are written in VRL (Vector Remap Language) and follow strict conventions for event transformation.

## Development Commands

### Testing

```bash
# Test all transformers
mise run :test

# Test specific transformer (from root)
mise run //github_events:test
mise run //argocd_notifications:test
mise run //kubewatch_cloudevents:test
mise run //passthrough:test

# Test from transformer directory
cd github_events && mise run :test

# Review and update expected outputs (use when inputs change or transformer logic is updated)
mise run //github_events:test -- --mode review
```

### Code Formatting

```bash
# Format all code (runs dprint and obfuscation)
mise run :format

# Obfuscate sensitive data in sample inputs
mise run :obfuscate
```

### CI Tasks

```bash
# Run complete CI suite
mise run :ci
```

## Architecture

### Repository Structure

Each transformer is a self-contained directory with:
- `transformer.vrl` - VRL transformation logic (main implementation)
- `cdviz-collector.toml` - Configuration example showing how to use the transformer
- `inputs/` - Sample input events (JSON files organized by event type)
- `outputs/` - Expected CDEvents output (used for testing)
- `mise.toml` - Test task definition
- `README.md` - Usage documentation

### VRL Transformers

Transformers convert source events to CDEvents using VRL. The pattern is:

1. Parse input event structure
2. Extract relevant data from source-specific fields
3. Map to CDEvents schema with proper field conventions
4. Preserve source-specific data in `customData.<source>` hierarchy

Example from github_events/transformer.vrl:
- Detects event type by checking for specific fields (e.g., `.body.package`, `.body.workflow_run`)
- Extracts timestamps and converts to ISO format
- Builds CDEvents JSON with proper `context`, `subject`, and `customData` structure
- Returns array of events (transformers can emit 0, 1, or multiple CDEvents per input)

### Testing Mechanism

The `cdviz-collector transform` command processes all JSON files in `inputs/` directory through the transformer and compares output against `outputs/` directory:

- `--mode check` (default): Fails if output doesn't match expected
- `--mode review`: Interactive mode to accept/reject changes

## CDEvents Field Conventions (Critical Rules)

### context.id
- Set to `"0"` (or omit) to enable automatic content-based ID generation by cdviz-collector
- DO NOT manually generate IDs or reuse source event IDs
- **Exception**: Keep `context.id` when the transformer is NOT creating a new CDEvent (filtering, normalizing, validating, or adding customData)
- This ensures reproducible, deterministic IDs based on event content

### context.source
- Use the URI of the cdviz-collector service that creates or modifies the event
- This identifies where the event was created/modified, not the original triggering system
- Value depends on cdviz-collector's running mode:
  - **`connect` mode (server)**: Use cdviz-collector URI with `source` as query parameter
  - **`send` mode**: Use URL of triggering system (pipeline, workflow, etc.)
  - **`transform` mode**: Use `http://cdviz-collector.example.com?source=cli-transform`
- cdviz-collector provides suggested value in metadata that transformers can use or override
- Customize using `http.root_url` in `cdviz-collector.toml`

### context.timestamp
- Extract timestamp from source event data when available (e.g., `.body.workflow_run.updated_at`)
- Parse and format as ISO 8601: `parse_timestamp(..., "%+")` then `format_timestamp!(..., format: "%+")`
- Avoid `now()` or automatic timestamps to ensure reproducible outputs for testing

### subject.id
- Use globally unique, hierarchical URI/URL identifying the subject entity
- Can be a URL, PURL, or absolute path starting with `/`
- Prefer API URIs over human-facing view URIs
- **DO NOT use `subject.source`** - make `subject.id` fully self-describing and globally unique
- The ID should work as a standalone identifier/reference in any context
- Examples:
  - Absolute paths: `/namespace/my-service`, `/cluster/us-1/staging`
  - API URLs: `https://github.com/org/repo/workflow/run`, `https://jenkins.example.com/job/job_name/`
  - For artifacts: Use PURL format (see artifactId section)

### subject.type
- Must match CDEvents subject types: `artifact`, `pipelineRun`, `taskRun`, `ticket`, `change`, `branch`, etc.

### environment.id
- Follow same rules as `subject.id` - it's a reference to an environment subject
- Often subjects don't know their environment, so this may need to be injected
- Define as absolute path starting with `/` for consistency
- Use hierarchical paths ordered from most to least stable: `/level/region/owner`
- Be consistent across all apps and configurations
- Examples: `/production`, `/pro/us-1/cluster-33`, `/staging`, `/dev/ephemeral-42`
- **Why**: Enables environment-level dashboards, filtering, and alerts

### artifactId
- Follow Package URL (PURL) specification: `pkg:type/namespace/name@version?qualifiers`
- Use appropriate type if supported, otherwise fallback to `generic`
- Common types: `oci`, `npm`, `maven`, `gem`, `nuget`, `github`
- For OCI images: Use image digest as version (NOT git commit SHA), include `repository_url` and `tag` as qualifiers
- Examples:
  - OCI: `pkg:oci/my-app@sha256:abc123...?repository_url=ghcr.io/myorg/my-app&tag=v1.2.3`
  - NPM: `pkg:npm/lodash@4.17.21`
  - Maven: `pkg:maven/org.springframework/spring-core@5.3.10`
  - Generic: `pkg:generic/my-app@1.2.3`

**Common PURL Pitfalls**:
- **Digest vs Tag**: Use image digest for immutability, NOT source code commit SHA
- **OCI Namespace**: `pkg:oci/` does NOT support namespace in path - use `repository_url` query parameter
- **Type-Specific Rules**: Each PURL type has unique encoding rules - consult the specification

### customData
- Preserve source-specific information not covered by CDEvents standard fields
- Structure as JSON object with source name at first level: `customData.github`, `customData.argocd`
- For webhook events, mirror original event structure (complete or filtered)
- Avoid duplicating data already in standard CDEvents fields
- Additional first-level keys may be added for information useful to other consumers

## Transformer Development Guidelines

### Metadata and Transformer Chaining

- Use `metadata` to transfer information between transformers
- Use `metadata` from extractors to initialize information
- Use the first transformer to initialize information when:
  - Not possible via extractor (pre-0.19)
  - Sharing information/transformers between multiple sources and transformer chains

### Adding a New Event Type to Existing Transformer

1. Add sample input JSON file to `inputs/<event-type>/`
2. Add expected output JSON file to `outputs/<event-type>/`
3. Update transformer.vrl with new event detection logic (typically an `else if exists(.body.field_name)` block)
4. Map source fields to CDEvents schema following field conventions from RULES.md
5. Extract timestamps from input data (avoid `now()` for reproducibility)
6. Run `mise run :test -- --mode review` to validate and accept outputs
7. Update transformer's README.md to document the new event type

### Creating a New Transformer

1. Create directory with transformer name
2. Add `transformer.vrl` with transformation logic
3. Add `cdviz-collector.toml` with configuration example
4. Create `inputs/` and `outputs/` directories with test cases
5. Add `mise.toml` with test task (copy pattern from existing transformers)
6. Add `README.md` with usage documentation
7. Update root README.md table with new transformer

### VRL Error Handling

- Use `!` suffix for functions that should fail loudly (e.g., `string!()`, `to_string!()`, `format_timestamp!()`)
- Use `??` operator for fallback values (e.g., `parse_timestamp(...) ?? now()`)
- Use `exists()` to check for optional fields before accessing them
- Filter null values from arrays: `filter(array) -> |_index, item| { !is_nullish(item) }`

## Important Files

- `RULES.md` - Comprehensive CDEvents field conventions (authoritative source for field usage)
- `AGENTS.md` - Quick reference for build/test commands and conventions
- `CONTRIBUTING.md` - Contribution workflow, prerequisites (mise, docker), commit signing requirements
- `mise.toml` - Root task definitions and tool dependencies
- `dprint.jsonc` - Code formatting configuration

## Technology Stack

- **VRL (Vector Remap Language)**: Transformation language for cdviz-collector
- **mise-en-place**: Task runner and environment manager
- **dprint**: Code formatter for JSON, Markdown, YAML
- **cdviz-collector**: Event collection and transformation tool

## Remote Transformer Usage

Transformers can be used remotely without cloning the repository by configuring cdviz-collector.toml:

```toml
[remote.transformers-community]
type = "github"
owner = "cdviz-dev"
repo = "transformers-community"

[transformers.github_events]
type = "vrl"
template_rfile = "transformers-community:///github_events/transformer.vrl"
```

## Commit Requirements

- All commits must be signed off: `git commit -s`
- Includes `Signed-off-by` line in commit message (required by CLA)
