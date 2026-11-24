# Loyal Spark: MCP servers and agent skill

[![Loyal Spark MCP connector](https://glama.ai/mcp/connectors/io.github.aspekt19/loyal-spark/badges/score.svg)](https://glama.ai/mcp/connectors/io.github.aspekt19/loyal-spark)

Onchain loyalty for AI agents on Base (chain 8453): create B20 loyalty programs, mint points to customer wallets, manage rewards, vouchers and gift certificates, trade loyalty tokens on a P2P market. Hosted service: nothing to install.

| Server | URL | Tools |
|---|---|---|
| Merchant | `https://api.loyalspark.online/loyalty-mcp` | 43 |
| Holder | `https://api.loyalspark.online/recipient-loyalty-mcp` | 20 |

Transport: Streamable HTTP (POST JSON-RPC). `tools/list` and `get_platform_info` need no key.

## Connect
```json
{"mcpServers":{"loyal-spark":{"url":"https://api.loyalspark.online/loyalty-mcp","headers":{"x-api-key":"lsk_..."}}}}
```
Get a merchant key (Free plan: 200 calls/month, all 5 scopes) for $0.01 USDC via x402: `POST https://api.loyalspark.online/x402-gateway/register-agent`. Or skip keys and pay per call (x402 v2, USDC on Base, $0.001–$0.01) at `https://api.loyalspark.online/x402-gateway/...`.

## Docs
- Agents guide: https://loyalspark.online/for-agents (no-JS: /for-agents.md)
- llms.txt: https://loyalspark.online/llms.txt · OpenAPI: https://loyalspark.online/openapi.json
- x402 discovery: https://api.loyalspark.online/.well-known/x402
- Skill: [skills/loyal-spark/SKILL.md](skills/loyal-spark/SKILL.md)

MIT licensed. The service source is private; this repo holds listing metadata (server.json, glama.json) and the agent skill.
