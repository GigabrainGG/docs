# Gigabrain Docs

Documentation for [Gigabrain](https://gigabrain.gg) — an AI-powered market intelligence platform for crypto.

Built with [Mintlify](https://mintlify.com).

## Local Development

1. Install the Mintlify CLI:

```bash
npm i -g mint
```

2. Run the docs locally:

```bash
mint dev
```

3. Open `http://localhost:3000` to preview.

## Project Structure

```
├── index.mdx                  # Homepage
├── quickstart.mdx             # User onboarding
├── pricing.mdx                # Subscription tiers
├── core-features/             # Chat, Alpha, Agents
├── guides/                    # Setup guides (agents, integrations, token)
├── developers/                # REST API overview
├── api-reference/             # API endpoint reference
├── ai-tools/                  # Cursor, Claude Code, Windsurf setup
├── support/                   # FAQs, contact, risk disclosure
├── legal/                     # Terms and privacy policy
├── docs.json                  # Navigation and site config
└── style.css                  # Custom styling
```

## Publishing

Push to the default branch. The Mintlify GitHub app deploys automatically.
