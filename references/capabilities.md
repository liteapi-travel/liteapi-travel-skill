# Other LiteAPI capabilities

Hotel search → prebook → pay → book is specified in [hotels.md](hotels.md). This file is the map for everything else.

Paths below are from the LiteAPI OpenAPI specs the MCP server loads. Do not invent request or response bodies. Before writing a call that is not in `hotels.md`:

1. Open the matching page under https://docs.liteapi.travel/reference/ (endpoint index: https://docs.liteapi.travel/reference/api-endpoints-overview).
2. Or list tools on the MCP server and read that tool's input schema: `https://mcp.liteapi.travel/api/mcp?apiKey=YOUR_API_KEY`.

Auth is the `X-API-Key` header on every module. Calls still go through your server.

## Hosts

| Host | Modules |
|---|---|
| `https://api.liteapi.travel/v3.0` | Static data, hotel rates, flights, experiences, eSIM, loyalty, prices, mapping, supply customization |
| `https://book.liteapi.travel/v3.0` | Hotel prebook, book, bookings |
| `https://da.liteapi.travel` | Vouchers, analytics |
| `https://pci-book.liteapi.travel` | Direct card payment only. Never send PAN or CVC to the other hosts. |

## Flights

Same shape as hotels: search → verify → prebook → book. Host is the API host, not the hotel book host.

| Call | Purpose |
|---|---|
| `POST /flights/rates` | Search by legs. |
| `POST /flights/rates/matrix` | Flexible-date price matrix. |
| `POST /flights/verify` | Verify an offer before prebook. |
| `POST /flights/prebooks` | Create the checkout session. |
| `GET /flights/prebooks/{prebookId}` | Read a prebook. |
| `POST /flights/prebooks/{prebookId}/services` | Attach seats or bags before book. |
| `POST /flights/bookings` | Complete the booking. |
| `GET /flights/bookings` | List. |
| `GET /flights/bookings/{bookingId}` | Retrieve. |
| `GET /flights/bookings/{bookingId}/cancellations` | Cancellation quote. |
| `POST /flights/bookings/{bookingId}/cancellations` | Cancel. |
| `GET /data/flights/airports` | Airport search. |
| `GET /data/flights/airlines` | Airline list. |

Payment on book follows the same rule as hotels: prefer the Payment SDK and `TRANSACTION_ID`. Read the flights book docs before using `CREDIT`, `CREDIT_CARD`, or `ACC_CREDIT_CARD`.

## Experiences (tours)

| Call | Purpose |
|---|---|
| `GET /experiences/tours` | Search. |
| `GET /experiences/tours/{id}` | Details. |
| `GET /experiences/tours/{id}/availability` | Availability. |
| `GET /experiences/tours/{id}/reviews` | Reviews. |
| `POST /experiences/tours/{id}/booking-options` | Options for a departure. |
| `POST /experiences/tours/{id}/prebooks` | Checkout session. |
| `POST /experiences/bookings` | Confirm. |
| `GET /experiences/bookings/{bookingId}` | Retrieve. |
| `GET /experiences/bookings/{bookingId}/cancel-preview` | Quote before cancel. |
| `POST /experiences/bookings/{bookingId}/cancel` | Cancel. |

## eSIM

`GET /data/esimply/destinations`, `GET /data/esimply/destinations/{destinationCode}/packages`, `GET /data/esimply/regions`, then `POST /esimply/orders/cart` and `POST /esimply/orders/submit`. Orders: `GET /esimply/orders`, `GET /esimply/orders/{iccid}`, usage `GET /esimply/usage/{iccid}`, top-up `POST /esimply/orders/submit-topup`.

## Loyalty and guests

| Call | Purpose |
|---|---|
| `GET /guests`, `GET /guests/{guestId}` | Guest records. |
| `GET /guests/{guestId}/bookings` | That guest's bookings. |
| `GET /guests/{guestId}/vouchers` | That guest's vouchers. |
| `GET /loyalties`, `PUT /loyalties` | Loyalty program settings. |
| `GET /guests/{guestId}/loyalty-points` | Point balance. |
| `POST /guests/{guestId}/loyalty-points/redeem` | Redeem points. |

## Vouchers

Host `https://da.liteapi.travel`: `POST /vouchers`, `GET /vouchers`, `GET /vouchers/{voucherID}`, `PUT /vouchers/{id}`, `PUT /vouchers/{id}/status`, `GET /vouchers/history`, `DELETE /vouchers/{id}/`. A voucher code can also be passed on hotel prebook as `voucherCode`. Confirm that field against the prebook docs before using it.

## Static hotel content

Host `https://api.liteapi.travel/v3.0`.

| Call | Purpose |
|---|---|
| `GET /data/places`, `GET /data/places/{placeId}` | Place search and lookup. |
| `GET /data/hotels` | Hotel list. |
| `GET /data/hotel` | One hotel (`hotelId` query). |
| `GET /data/reviews` | Hotel reviews. |
| `GET /data/countries`, `GET /data/cities` | Geography. |
| `GET /data/currencies` | Currencies. |
| `GET /data/iataCodes` | IATA codes. |
| `GET /data/facilities` | Facilities. |
| `GET /data/hotelTypes` | Hotel types. |
| `GET /data/chains` | Chains. |
| `GET /data/languages` | Languages. |
| `GET /data/weather` | Weather for a destination. |
| `GET /data/hotels/semantic-search`, `GET /data/hotel/search` | Semantic hotel search. |
| `GET /data/hotel/ask` | Question about one hotel. |

Prefer `POST /hotels/rates` with `aiSearch` (see [hotels.md](hotels.md)) when the user wants prices plus hotel cards in one call. Semantic search does not return live rates.

## Rates extras, prices, analytics

| Call | Host | Purpose |
|---|---|---|
| `POST /hotels/min-rates` | API | Cheapest rate per hotel. |
| `POST /hotels/rates/batch` | API | Rates across check-in dates. Enterprise. |
| `GET /prices/city`, `GET /prices/hotels`, `GET /prices/public` | API | Price index, not a bookable offer. |
| `POST /analytics/weekly`, `/analytics/report`, `/analytics/markets`, `/analytics/hotels` | `da.liteapi.travel` | Account analytics. |
| `GET /bookings/guest-nationality-report`, `/bookings/source-markets-report`, `/bookings/hotels-sales-report` | `da.liteapi.travel` | Booking reports. |

Analytics and voucher routes are account data. Do not expose them in a public whitelabel without an authenticated admin.

## Hotel booking extras

On `https://book.liteapi.travel/v3.0`, beyond the flow in [hotels.md](hotels.md):

| Call | Purpose |
|---|---|
| `GET /prebooks/{prebookId}` | Read a prebook. |
| `PUT /bookings/{bookingId}/amend` | Amend the guest name. |
| `POST /bookings/{bookingId}/alternative-prebooks` | Amend dates or occupancies. |
| `POST /rates/rebook` | Hard amendment (rebook). |

Read the docs for each before calling. Amend and rebook move money.

## Mapping and supply customization

Partner hotel-id mapping (`/hotels/match/...`), room mapping (`/rooms/match`, `/rooms/group`), and `GET`/`PUT /supply-customization` are account configuration, not guest-facing booking. Skip them unless the user is building an internal mapping or supply tool.
