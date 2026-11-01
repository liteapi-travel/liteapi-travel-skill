---
name: liteapi-travel
description: >-
  Builds travel booking experiences on the LiteAPI (Nuitee Connect) REST API:
  hotel search, rates, prebook, payment, and booking, plus flights, experiences,
  and other LiteAPI products. Use when building a hotel or travel booking site,
  whitelabel, agent, or chatbot, or when the user mentions LiteAPI, Nuitee
  Connect, hotel rates, prebook, flights, or booking flows.
---

# LiteAPI travel experiences

Build the app against the REST API. Call every LiteAPI endpoint from server-side code. The API key must never reach the browser, and the endpoints reject browser calls (CORS).

Read [references/hotels.md](references/hotels.md) before writing a hotel booking flow. Read [references/capabilities.md](references/capabilities.md) for flights, experiences, eSIMs, loyalty, vouchers, and static data. Read [references/security.md](references/security.md) before choosing where the key lives.

## Key and hosts

Read the key from the server env var `LITEAPI_API_KEY`. Do not hardcode it. Do not commit it.

| Prefix | Environment | Payment SDK `publicKey` |
|---|---|---|
| `prod_` | Production | `live` |
| `sand_` or `sandbox_` | Sandbox | `sandbox` |

The API key environment and the Payment SDK `publicKey` must match. Start on a sandbox key.

| Host | Use |
|---|---|
| `https://api.liteapi.travel/v3.0` | Hotel content, rates, flights, experiences, loyalty, static data |
| `https://book.liteapi.travel/v3.0` | Hotel prebook, book, list, retrieve, cancel |

Headers on every request:

```
X-API-Key: <LITEAPI_API_KEY>
accept: application/json
content-type: application/json   # POST and PUT with a body
```

Set the HTTP client timeout to at least 15 seconds. Rate search is often 6–12 seconds. Prebook and book are often 5–10 seconds. Show a loading state. On HTTP 429, back off and retry.

## Hotel booking flow

Default product unless the user asks for something else. Full request and response shapes are in [references/hotels.md](references/hotels.md).

1. **Search.** Destination autocomplete: `GET /data/places?textQuery={q}` on the API host. Keep `placeId` and `displayName`. Free-text "search by vibe": skip places and send `aiSearch` on the rates call.
2. **Rates.** `POST /hotels/rates` with `checkin`, `checkout`, `currency`, `guestNationality`, `occupancies`, and one of `placeId`, `hotelIds`, or `aiSearch`. Send `roomMapping: true`. On a results list, send `maxRatesPerHotel: 1` and `includeHotelData: true`.
3. **Hotel page.** `GET /data/hotel?hotelId={id}`. Re-query rates for that `hotelId` **without** `maxRatesPerHotel` so every offer comes back. Group offers by `mappedRoomId`. Show the room name and its first photo.
4. **Prebook.** `POST /rates/prebook` on the book host with `offerId` (from `roomTypes[].offerId`) and `usePaymentSdk: true`. This can fail because the rate sold out — send the user back to the hotel page to pick another rate. Persist `prebookId`, `transactionId`, and `secretKey` together. A later prebook replaces all three, and they cannot be fetched again after the payment redirect.
5. **Guests, then payment.** Collect holder and guest first name, last name, and email before showing the payment form. Load `https://payment-wrapper.liteapi.travel/dist/liteAPIPayment.js`. Pass `secretKey` from prebook, `publicKey` of `sandbox` or `live`, and a `returnUrl` on your own origin that still has `prebookId` and `transactionId`. In sandbox, tell the user to pay with card `4242424242424242`, any CVC, and any future expiry.
6. **Book.** After the SDK redirects to `returnUrl`, `POST /rates/book` with `prebookId`, `holder`, `guests`, and `payment.method: "TRANSACTION_ID"` plus the stored `transactionId`. Show `bookingId` and `hotelConfirmationCode`.
7. **Manage.** `GET /bookings` (by `guestId` or `clientReference`), `GET /bookings/{bookingId}`, `PUT /bookings/{bookingId}` to cancel. All on the book host.

## Fields to render

| Field | Use |
|---|---|
| `roomTypes[].offerId` | The id you prebook. Not `rateId`. |
| `rates[].name` | Room rate name. |
| `rates[].mappedRoomId` | Group rates that belong to the same room. |
| `rates[].boardName` | Meal plan label. |
| `retailRate.total[0]` | Price and currency to display. |
| `taxesAndFees[].included` | Whether taxes are included. |
| `cancellationPolicies.refundableTag` | `RFN` refundable, `NRFN` non-refundable. |
| `bookingId`, `hotelConfirmationCode` | Confirmation screen. |

For an `aiSearch` response, render cards from the `hotels[]` array (name, `main_photo`, address, rating) and attach the price from `data[]` where `hotelId` matches `hotels[].id`.

## Other products

Do not invent request bodies. Hotels are fully specified in [references/hotels.md](references/hotels.md). For flights, experiences, eSIMs, loyalty, vouchers, prices, and static data, follow the path list in [references/capabilities.md](references/capabilities.md), then confirm the body against the docs or the live MCP tool schema before coding it.

## Optional MCP

The same API is exposed as MCP tools for the agent to discover or call while building. Do not put the MCP URL in client-side code.

```
https://mcp.liteapi.travel/api/mcp?apiKey=YOUR_API_KEY
```

Prefer writing a server-side REST integration into the user's app. Use MCP to look up a tool's schema when the REST body is not in these references.

## Docs

- Booking wiring this skill is based on: https://docs.liteapi.travel/reference/prompt-for-vibe-coding-tools
- Endpoint index: https://docs.liteapi.travel/reference/api-endpoints-overview
- MCP setup: https://docs.liteapi.travel/reference/mcp-server
