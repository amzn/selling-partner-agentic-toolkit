# Manage Agents — granting AI-agent access to a secondary user

Granting a secondary user access to the Selling Partner plugin is done in **Seller Central**, at
**Manage Your Permissions → Manage Agents** (alongside Manage Your Apps and Manage Services). There
is no SP-API/MCP tool for this — it is a UI action taken by the account owner, an administrator, or
a permissions manager.

## Who can do this
- The **account owner**, an **administrator**, or a **permissions manager** of the Seller Central
  account.
- The account administrator and any permissions managers **always** have access to authorize
  agents. That access is on by default and cannot be turned off.
- A secondary user without those roles cannot grant themselves access — they must ask an
  administrator.

## Two settings work together

| Setting | What it controls |
|---|---|
| **Agent enablement** | Whether an agent, such as the Selling Partner plugin, is enabled for your account at all. |
| **Secondary user authorization** | Which secondary users are allowed to authorize the enabled agents for their own use. |

An agent can be enabled for your account while **no** secondary users are permitted to authorize
it. Both settings have to be right: enabling the agent alone does not let secondary users in, and
granting secondary users while the agent is disabled has no effect.

## Steps (account owner / administrator / permissions manager)
1. Sign in to **Seller Central**.
2. Go to **Manage Your Permissions → Manage Agents**.
3. **Enable the agent** you want to make available for your account.
4. **Choose who can authorize it** — grant **all secondary users at once**, or **select individual
   users**.

## What this grant does — and doesn't do
- **Does:** allow that secondary user to **authorize the Selling Partner plugin** in their AI
  assistant, scoped to their Seller Central roles. Once granted, they can also **disconnect any
  agent they have connected**, from their own Manage Agents view.
- **Doesn't:** connect the plugin for them. After you grant access, the secondary user must still,
  on their own side:
  1. Install/open the plugin in their assistant (Claude or Quick),
  2. **Sign in with their own Amazon selling account**, and
  3. Authorize access when prompted.
- Write actions the agent takes still require **approval** by default, per the plugin's
  human-in-the-loop model.

## Before the grant exists — what the secondary user sees
A secondary user who tries to connect before you have granted them cannot complete authorization.
They see a screen telling them to ask an account administrator to enable the Amazon Selling Partner
plugin for them. **That screen is expected behavior, not an error** — it is not a connection
failure and should not be troubleshooted as one.

## Revoking access
- **To remove one user's access:** remove them from **secondary user authorization**.
- **Do not** turn off **agent enablement** to offboard a single person — that disables the plugin
  for the **entire account**.
- A secondary user can also disconnect an agent they connected themselves.

## Good practice
- Grant access only to **trusted** users — AI-agent access lets that user act on the account
  through the plugin.
- Changes can take a few minutes to apply.
- When onboarding a whole team, use the **grant all secondary users** option rather than adding
  users one at a time.
- Keep a record of who you granted access to, and remove authorization when someone no longer
  needs it.

> Source: Seller Central Help — "Selling Partner plugin › Managing access for secondary users."
