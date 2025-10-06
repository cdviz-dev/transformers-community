# Agent Guidelines for transformers-community

## Build/Test Commands
- Run all tests: `mise run test`
- Run single transformer test: `mise run test:transform:<name>` (e.g., `mise run test:transform:github_events`)
- Review and update outputs: `mise run test:transform:<name> -- --mode review`
- Format code: `mise run format` (runs dprint and obfuscation)
- CI: `mise run ci`

## Code Style
- **Language**: VRL (Vector Relic Language) for transformers
- **Indentation**: 2 spaces (JSON/YAML/TOML), 4 spaces (VRL/code blocks per .editorconfig)
- **Line endings**: LF (Unix-style)
- **Trailing whitespace**: Remove
- **Final newline**: Required
- **Max line length**: Context-dependent (see .editorconfig)
- **Formatting**: Use `dprint fmt` for JSON, Markdown, YAML files

## Transformer Conventions
- Set `context.id = "0"` to let collector auto-generate IDs
- Use `context.source = "/<source>"` pattern (e.g., `/github`, `/argocd`)
- Format `subject.id` as global identifier (e.g., `namespace/name` or API URL)
- Keep `subject.source` empty (all info in `subject.id`)
- Preserve source-specific data in `customData.<source>` hierarchy
- Use PURL format for `artifactId` fields
- Chain transformers for separation of concerns (metadata injection → business logic)