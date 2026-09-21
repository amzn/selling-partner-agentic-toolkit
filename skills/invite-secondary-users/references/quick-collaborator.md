# Quick only — invite the user as a collaborator on your Quick account

**This applies only when the plugin is used in Amazon Quick.** In Claude (or any non-Quick
assistant) there is no collaborator step — skip it.

## Why there are two grants on Quick
For a secondary user to use the Selling Partner plugin **inside Quick**, they need **both**:
1. **Seller Central — Manage Agents:** AI-agent access enabled (see
   [manage-agents-guide.md](manage-agents-guide.md)). This is the Amazon-side authorization.
2. **Quick — collaborator:** access to **your Quick account**, because the plugin and its skills
   are installed/used within a Quick account. The Manage Agents grant does not by itself give the
   user access to your Quick workspace.

If you grant only Manage Agents access but never add the user to Quick, they won't be able to use
the plugin in Quick. If you add them to Quick but skip Manage Agents, they can't authorize the
plugin against your Seller Central account. **Both are required on Quick.**

## Steps (Quick account owner)
1. Open **Amazon Quick** (desktop app).
2. Invite the secondary user as a **collaborator** on your Quick account (use the same person /
   email you granted on Manage Agents).
3. Have the user **accept the collaborator invite**, then **sign in to the plugin with their own
   Amazon selling account** to authorize access.

## Note
- Quick subscription and Selling Partner plugin access activate on their own schedules; the user
  needs plugin access active on their account too.
- Exact Quick menu labels can vary as Quick evolves — the requirement is: the user must be a
  collaborator on your Quick account **in addition to** the Manage Agents grant.
