Loyal Spark is a hosted remote MCP server; do not clone or build anything.
1. Add to the MCP client config: {"mcpServers":{"loyal-spark":{"url":"https://api.loyalspark.online/loyalty-mcp","headers":{"x-api-key":"<lsk_ key>"}}}}
2. No key yet? tools/list works without one. To get a key, the user's wallet pays $0.01 USDC on Base via x402 at POST https://api.loyalspark.online/x402-gateway/register-agent: ask the user before paying.
3. Start with GET https://api.loyalspark.online/agent-api/workflow/program-status and follow next_actions.
