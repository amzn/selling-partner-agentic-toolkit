# Amazon Selling Partner Plugin — v1.0.0

Skills for managing an Amazon seller account via SP-API, through the Amazon Selling Partner Connector (MCP). Write actions are always human-in-the-loop (drafted for your approval).

## Connector

The bundled `.mcp.json` registers the Amazon Selling Partner Connector (MCP server) at
`https://sellingpartner-ai.amazon.com/mcp`. You authorize
access through the standard Seller Central OAuth consent flow; the assistant can only use
tools allowed by your Seller Central roles. Write actions are drafted for your approval.

## Skills

- **seller-analytics** — seller performance metrics (inventory, traffic, sales); discovers metric IDs before querying.
- **stockout-prevention** — quantify stockout risk from days-of-supply and velocity, recommend how much to reorder and by when, and check the inbound-shipment gap. Gives no pricing advice in either direction; if you name a specific price it will draft that exact change for your approval.
- **fba-inbound-management** — full FBA inbound plan workflow (create -> packing -> placement -> transportation -> confirm) with approval gates.
- **listing-troubleshooter** — the router for "something's wrong with my listing"; scans status/issues and routes to the right specialist.
- **listing-issues** — diagnose and fix reported listing issues (suppressions, attribute errors) with preview -> approve.
- **listing-buyability** — why a listing isn't buyable (offer, completeness, inventory) and fix the offer with approval.
- **listing-searchability** — why a listing isn't found + search-content optimization with approval.
- **listing-compliance** — a pre-flight compliance gate run before any compliance-sensitive listing write (claims, ingredients, category, condition, images, identifiers); advisory, never writes.
- **invite-secondary-users** — guide the account admin to grant AI-agent access to secondary users via Seller Central Manage Agents (advisory; UI action).
- **support-case-helper** — route the seller to Amazon Seller Support and prepare a clean, PII-free case summary; never submits without explicit confirmation.
- **sp-api-knowledge** — answer SP-API developer questions and guide integration design, grounded in the SP-API documentation.

## Commands

- `/sp-stockout-check` — run the stockout-prevention workflow.
- `/sp-fix-listing` — find and fix suppressed/inactive listings.
- `/sp-sales-drop` — diagnose why sales dropped and get safe next steps.

## Data sent

When you use these skills, the assistant sends requests to the Amazon Selling Partner Connector (MCP) at (`https://sellingpartner-ai.amazon.com/mcp`) over an authenticated connection:

- **What is sent:** the tool calls the skills make on your behalf (e.g. requests for your
  inventory, sales/traffic analytics, listing status, or a drafted listing/price change) and
  the parameters those calls need (such as your Merchant Token / marketplace and the SKU/ASIN
  in question). Your prompt text is processed by your AI assistant per its own terms.
- **What comes back:** your Amazon business data (inventory, analytics, listing details) that
  the gateway returns for the calls you approved.
- **Authorization & scope:** access is granted through the Seller Central OAuth consent flow
  and is limited to the tools your Seller Central roles allow. Write actions are never executed
  without your explicit approval.
- **Not sent by these skills:** the skills do not transmit data to any non-Amazon endpoint,
  and do not persist your account identifiers or data beyond the session. Some skills
  (e.g. support-case-helper) explicitly redact PII from any summary they produce.
- **Privacy:** data handled by Amazon is subject to the
  [Amazon.com Privacy Notice](https://www.amazon.com/gp/help/customer/display.html?nodeId=GX7NJQ4ZB8MHFRNJ).
  Data processed by your AI assistant is subject to that assistant's own terms and privacy policy.
