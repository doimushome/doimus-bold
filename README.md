# doimus-bold

Doimus native plugin for Bold Smart Locks. A Bold Connect hub is required.

## Features

- Lock/unlock control via remote activation
- Auto-relock after device's activation timeout
- Bold Connect hub shown as switch (or lock via config)
- Automatic access token refresh
- Polls device list every 24h

## Configuration

> **Note:** The recommended authentication method is the OAuth flow via the Doimus hub (see "OAuth Authentication Flow" above). The `accessToken`/`refreshToken` fields are only needed as a fallback for legacy setups.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `accessToken` | string | — | Bold API access token (legacy fallback — use OAuth flow instead) |
| `refreshToken` | string | — | Bold API refresh token (legacy fallback — use OAuth flow instead) |
| `refreshURL` | string | *built-in* | Custom refresh URL (for custom auth backend) |
| `legacyAuthentication` | boolean | `false` | Use legacy OAuth authentication |
| `showControllerAsLock` | boolean | `false` | Show Bold Connect hub as lock instead of switch |

## Device Capabilities

Locks expose:
| Capability | Description |
|------------|-------------|
| `locked` | `true` = locked, `false` = unlocking |
| `active` | `true` while lock is activated/unlocked |

Bold Connect hubs expose:
| Capability | Description |
|------------|-------------|
| `on` | `true` when activated |

## OAuth Authentication Flow

This plugin uses OAuth2 to authenticate against the Bold API. The flow is brokered by the Doimus hub:

1. **User taps "Authenticate"** on the plugin config screen in the mobile app.
2. **Mobile app calls hub** → `GET /api/v1/auth/authorize-url?plugin_id=doimus-bold`
3. **Hub generates an OAuth authorize URL** with `redirect_uri=https://doimus.com/callback` and a CSRF `state` nonce.
4. **System browser opens** the Bold IdP authorization page.
5. **User logs in** to their Bold account and grants access.
6. **Bold IdP redirects** to `https://doimus.com/callback?code=xxx&state=yyy`.
7. **doimus.com/callback** is a static page that immediately redirects to `doimus://auth/callback?code=xxx&state=yyy` via JavaScript `window.location.replace()`.
8. **Mobile app intercepts** the `doimus://` deep link and parses the auth code and state.
9. **Mobile app calls hub** → `POST /api/v1/auth/oauth/exchange` with `{code, state}`.
10. **Hub validates the state nonce**, exchanges the code with Bold's `/v2/oauth/token`, and stores the encrypted tokens in SQLite.
11. **Plugin sandbox** receives tokens on next start via `api.getOAuthToken()`.
12. **Token refresh** is handled transparently by the hub — the plugin never manages tokens directly.

### Configuration

The OAuth provider config lives in `package.json` under `doimus.oauth`:

| Field | Description |
|-------|-------------|
| `authorization_url` | Bold IdP authorization page |
| `token_url` | Bold token exchange endpoint |
| `client_id` | Bold OAuth client ID |
| `client_secret` | Bold OAuth client secret |
| `redirect_uri` | Public HTTPS URL that forwards to `doimus://auth/callback` |
| `scopes` | OAuth scopes (empty = default) |

### Token Resolution

The plugin's `resolveTokens()` uses this priority:
1. `api.getOAuthToken()` — tokens injected by the hub (from the OAuth flow)
2. `accessToken` / `refreshToken` config fields — manual fallback for legacy setups

## Official Bold API

This plugin integrates with the Bold Smart Lock platform API. For reference when extending or debugging this plugin, see the official documentation:

- [Bold Integration Guide](https://sesamsolutions.gitlab.io/public-documentation/integration/)
- [Bold API Reference](https://apidoc.boldsmartlock.com)

Key endpoints used by this plugin:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/effective-device-permissions` | GET | List accessible devices |
| `/v1/devices/{id}/remote-activation` | POST | Unlock/activate a lock |
| `/v2/oauth/token` | POST | OAuth token refresh (legacy flow) |

Authentication requirements per the official docs:

- Remote activation requires a `user`-level session with the `activate` scope
- Tokens should be refreshed before expiration; always handle `401` responses
- Organization-level sessions can use API keys (Basic Auth) but cannot activate locks directly

## Credits

This plugin is a port of [homebridge-bold](https://github.com/StefanNienhuis/homebridge-bold) by [Stefan Nienhuis](https://github.com/StefanNienhuis). Thanks to Erik Nienhuis for reverse-engineering the Bold API.

## License

MIT
