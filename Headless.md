https://developer.salesforce.com/blogs/2026/05/connect-claude-with-salesforce-hosted-mcp-servers

Instructions for Claude Code
1. Run the following command in a terminal where:
– MY_MCP_SERVER_NAME is replaced by the name of the server that you’re connecting to. You’re free to enter any value, but we recommend that you stick to the following convention: salesforce- followed by the API name of the MCP server. For example: salesforce-sobject-all for the sobject-all server.
–MY_MCP_SERVER_URL is replaced by the MCP server URL that you copied earlier. Pay attention to the fact that server URLs differ depending on whether you are connecting to a production or sandbox/scratch org.
–MY_ECA_CONSUMER_KEY is replaced by the ECA consumer key you’ve obtained in the previous step.

claude mcp add --transport http MY_MCP_SERVER_NAME MY_MCP_SERVER_URL --callback-port 38000 --client-id "MY_ECA_CONSUMER_KEY" --client-secret

For example, to install the sobject-all MCP server from a scratch org, run:
claude mcp add --transport http salesforce-sobject-all https://api.salesforce.com/platform/mcp/v1/sandbox/platform/sobject-all --callback-port 38000 --client-id "XXX" --client-secret


sobject-all:
claude mcp add --transport http salesforce-sobject-all https://api.salesforce.com/platform/mcp/v1/platform/sobject-all --callback-port 38000 --client-id "3MVG91oqviqJKoEEU2cK9YykyCiHb3PmP1ebKFotxsCp5I.TpNpjbZA_C64Wo_16r8SfLEPZu9KjjbuvwGbAc" --client-secret

CustomSalesTools:
claude mcp add --transport http CustomSalesTools https://api.salesforce.com/platform/mcp/v1/custom/CustomSalesTools --callback-port 38000 --client-id "3MVG91oqviqJKoEEU2cK9YykyCiHb3PmP1ebKFotxsCp5I.TpNpjbZA_C64Wo_16r8SfLEPZu9KjjbuvwGbAc" --client-secret


2. When prompted to do so, paste the ECA consumer secret you’ve obtained previously.
3. Launch Claude Code with the claude terminal command.
4. Run /mcp to list the MCP servers.
5. Select your newly added MCP server from the list (it is annotated with a “needs authentication” message).
6. Select Authenticate.
7. When prompted, allow access to the MCP server.