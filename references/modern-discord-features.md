# Modern Discord Features and Invariants

## Contents

- User-Installable Apps (Integration Types & Contexts)
- Discord Activities & Embedded App SDK
- AutoMod Architecture and ReDoS Prevention
- Components V2 & Rich Interactions

---

## User-Installable Apps (Integration Types & Contexts)

Discord supports installing apps directly to user profiles, allowing commands to run in any server, Group DM, or Direct Message—even where the bot is not a member of the guild.

### Key Enums & Configuration

1. **Integration Types** (`integration_types`):
   - `0` (`GUILD_INSTALL`): Traditional bot installation into a guild by a server admin.
   - `1` (`USER_INSTALL`): App installed directly to a user's account.

2. **Interaction Contexts** (`contexts`):
   - `0` (`GUILD`): Allowed in Discord servers.
   - `1` (`BOT_DM`): Allowed in direct messages with the bot.
   - `2` (`PRIVATE_CHANNEL`): Allowed in group DMs and DMs between users.

### Architectural Invariants for User Apps

- **Null Guild Invariant**: In user-installed contexts outside normal guilds (`contexts: [1, 2]`), `interaction.guildId` (JS) / `interaction.guild` (Py) is `null`/`None`. Code MUST NOT assume `guild` or `member` exists.
- **Author Identity**: Prefer `interaction.user` over `interaction.member.user`. In external guilds, `interaction.member` contains only author-level context, not full guild member caches.
- **Permission Scoping**: Guild administrative permissions (e.g. `ManageGuild`, `BanMembers`) cannot be verified in external guilds. Only execute guild-scoped actions when `interaction.guildId` is valid and verified.
- **Default Ephemerality**: In foreign servers where the bot is not officially installed, slash command responses SHOULD default to ephemeral (`ephemeral: true`) unless the interaction was explicitly designated for public channel sharing by user action.

---

## Discord Activities & Embedded App SDK

Discord Activities run web applications inside an iframe within Discord Voice Channels, utilizing `@discord/embedded-app-sdk`.

### Security & Lifecycle Invariants

- **Token Exchange Security**: The web client initiates authorization with `discordSdk.commands.authorize()`. The returned code must be exchanged for an access token on the bot's secure backend via OAuth2 token exchange (`/api/oauth2/token`). NEVER expose client secrets to the iframe frontend.
- **CSP & Frame Ancestors**: Activity frontend servers must declare Content Security Policy (CSP):
  ```http
  Content-Security-Policy: frame-ancestors https://discord.com https://*.discord.com;
  ```
- **Voice State & Participant Lifecycle**: Track participant joins and leaves via voice state updates (`voiceStateUpdate`). Tear down active game/activity sessions and clean up allocated backend state when all participants leave.
- **State Synchronization**: WebSocket or state channels between activity clients must authenticate using validated Discord user IDs tied to the active voice session.

---

## AutoMod Architecture and ReDoS Prevention

Automated Moderation (AutoMod) rules enforce content policies natively at the Discord gateway.

### Invariants & Protection

- **ReDoS Prevention**: When compiling user-supplied regex patterns for custom AutoMod rules, patterns MUST be validated against catastrophic backtracking (ReDoS). Enforce character class limits and reject unanchored nested quantifiers (e.g., `(a+)+`).
- **Audit Attribution**: All automated moderation actions (deleting messages, sending alerts, issuing timeouts) must include clear audit log reason metadata attributing the triggering rule and matched pattern type.
- **Rule Limits**: Guilds have strict limits on active AutoMod rules per trigger type. Verify rule count capacity before creating new rules programmatically.

---

## Components V2 & Dynamic Interfaces

- **State Preservation**: Component callbacks must survive process restarts by serializing state identifiers inside `custom_id` payloads or storing them in a persistent cache with strict TTL.
- **Single Owner Invariant**: Action rows containing sensitive operations (confirmations, payments, ticket closure) must bind to the original actor ID and reject interaction attempts from third-party users with an ephemeral warning.
