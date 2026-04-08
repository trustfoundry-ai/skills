# TrustFoundry Agent Skills

Agent-native integration files for the [TrustFoundry](https://trustfoundry.ai) legal research API.

## Quick Install

Works with **Claude Code**, **OpenAI Codex**, **GitHub Copilot**, and any [agentskills.io](https://agentskills.io)-compatible agent:

```bash
npx skills add trustfoundry-ai/skills
```

## What's Included

### `trustfoundry-legal-search/`

Integration skill for the TrustFoundry legal research API. Provides:

- Authentication patterns (API key via `X-API-Key` header)
- All 4 public endpoint contracts (agentic search, direct search, result retrieval, document description)
- NDJSON streaming parser guidance
- Error handling for rate limits (429), quota (429), and credit exhaustion (402)
- TypeScript and Python code examples
- Cursor and Windsurf IDE rule configurations

### Structure

```
trustfoundry-legal-search/
├── SKILL.md                           # Main skill (auth, endpoints, streaming, errors)
└── references/
    ├── endpoints.md                   # Detailed endpoint contracts and schemas
    ├── examples.md                    # Full TypeScript and Python examples
    ├── cursor-windsurf.md             # IDE rule documentation
    ├── trustfoundry.mdc               # Standalone Cursor rule (download directly)
    └── trustfoundry.windsurfrules     # Standalone Windsurf rule (download directly)
```

## Manual Installation

If you prefer not to use `npx skills`:

**Claude Code / Codex / Copilot:**
```bash
# Copy the skill directory to your project
cp -r trustfoundry-legal-search/ .claude/skills/trustfoundry-legal-search/
# Or for Codex:
cp -r trustfoundry-legal-search/ .agents/skills/trustfoundry-legal-search/
# Or for Copilot:
cp -r trustfoundry-legal-search/ .github/skills/trustfoundry-legal-search/
```

**Cursor:**
See `references/cursor-windsurf.md` for the `.mdc` file to copy into `.cursor/rules/`.

**Windsurf:**
See `references/cursor-windsurf.md` for the `.windsurfrules` file.

## Getting Started

1. Sign up at [trustfoundry.ai](https://trustfoundry.ai)
2. Go to **Settings > Developers > API Keys** and create a key
3. Set `TRUSTFOUNDRY_API_KEY` in your environment
4. Install this skill and start building

## Links

- [API Documentation](https://api.trustfoundry.ai)
- [llms.txt](https://api.trustfoundry.ai/llms.txt) — Agent discovery file
- [llms-full.txt](https://api.trustfoundry.ai/llms-full.txt) — Full integration guide
- [OpenAPI Spec](https://api.trustfoundry.ai) — Swagger UI

## License

MIT
