# Security

## Server-side only

Every LiteAPI request runs on the server (route handler, serverless function, or backend). The browser never calls `api.liteapi.travel`, `book.liteapi.travel`, `da.liteapi.travel`, or `pci-book.liteapi.travel`.

Two reasons:

- The API key is a secret. A key in client JavaScript, a mobile bundle, or a public repo can book and spend.
- The endpoints are not CORS-enabled for browser apps. A client-side `fetch` fails even with a key.

Proxy pattern: the UI calls your own origin (`/api/rates`, `/api/prebook`, `/api/book`). That handler attaches `X-API-Key` from the environment and forwards the body.

For a static host (Lovable, v0, Cursor static preview), add server functions (edge functions, Next.js route handlers, or similar) before wiring the API. Do not "temporarily" call LiteAPI from the client.

## Key storage

- Env var name: `LITEAPI_API_KEY`.
- Production: a secret manager or the host's encrypted env. Not a committed `.env`.
- Sandbox keys (`sand_` / `sandbox_`) are fine for local development. Still do not commit them.
- Production keys start with `prod_`. Never put a `prod_` key in a sample, a screenshot, a log, or a client bundle.
- Do not put the key in a query string on your own app. LiteAPI's MCP URL is the exception documented by LiteAPI (`?apiKey=`). Do not copy that pattern into the app you generate.

If the user pastes a key into chat, use it only as the env value and do not echo it back in code, commits, or README.

## Sandbox vs production

The key prefix and the Payment SDK `publicKey` are a pair:

| Key prefix | SDK `publicKey` |
|---|---|
| `sand_` or `sandbox_` | `sandbox` |
| `prod_` | `live` |

A sandbox key with `publicKey: "live"` (or the reverse) breaks payment. Default new projects to sandbox. Switch both when the user asks to go live.

Sandbox card: `4242424242424242`, any CVC, any future expiry. Show that hint only when the key is a sandbox key.

## Payment secrets

Prebook returns `secretKey` and `transactionId`. They are payment credentials.

- Do not log them.
- Persist the `prebookId` + `transactionId` pair in your server session or database, not only in a query string, if the booking page is shared or logged.
- The Payment SDK has to receive `secretKey` in the browser. That is the SDK's model. It is not a reason to also expose `LITEAPI_API_KEY`.
- Card numbers and CVC go to the Payment SDK (Stripe Elements) or, if the user explicitly uses direct card payment, to `https://pci-book.liteapi.travel`. Never send PAN or CVC to the API or book hosts, and never store them.

## What the generated app must not do

- Book, cancel, amend, or rebook from a public unauthenticated endpoint without your own check (a signed return token, a session, or an idempotency key you issued at prebook).
- Trust a client-supplied price. Charge and confirm from the prebook response, not from a hidden field the browser can edit.
- Log `X-API-Key`, `secretKey`, or raw card data.
- Retry book blindly. A timeout after `POST /rates/book` may have succeeded. Retrieve by `clientReference` or booking id before creating another booking.

## Rate limits

Default limit is 500 requests per second, then HTTP 429. Usage is billed per the account plan. Back off on 429. Do not fan out unbounded rate searches (one search per keystroke). Debounce place autocomplete and cache static data (`/data/countries`, `/data/currencies`, facilities) for hours.

## HTML from the API

`hotelDescription`, `hotelImportantInformation`, and similar fields are HTML. Sanitize before rendering, or strip tags. Do not pass them through unsanitized `dangerouslySetInnerHTML` / `innerHTML`.
