# Hotel booking wiring

Server-side only. Headers on every call: `X-API-Key`, `accept: application/json`, and `content-type: application/json` on POST.

- Content and rates: `https://api.liteapi.travel/v3.0`
- Prebook, book, bookings: `https://book.liteapi.travel/v3.0`

Flow: **places (optional) → rates → hotel details → prebook → payment SDK → book**.

## 1. Search places

`GET /data/places?textQuery={q}`

Autocomplete. Keep `placeId` and `displayName`.

```json
{
  "data": [
    {
      "placeId": "ChIJdd4hrwug2EcRmSrV3Vo6llI",
      "displayName": "London",
      "formattedAddress": "UK"
    }
  ]
}
```

## 2. Search hotel rates

`POST /hotels/rates`

### By place

```json
{
  "occupancies": [{ "adults": 2 }],
  "currency": "USD",
  "guestNationality": "US",
  "checkin": "YYYY-MM-DD",
  "checkout": "YYYY-MM-DD",
  "placeId": "<from places>",
  "roomMapping": true,
  "maxRatesPerHotel": 1,
  "includeHotelData": true
}
```

### Single hotel (results list)

Same body, with `hotelIds: ["<hotelId>"]` instead of `placeId`.

### Response

```json
{
  "data": [
    {
      "hotelId": "<id>",
      "roomTypes": [
        {
          "offerId": "<offerId>",
          "rates": [
            {
              "name": "<room>",
              "mappedRoomId": 123,
              "boardName": "Room Only",
              "retailRate": {
                "total": [{ "amount": 131.54, "currency": "USD" }],
                "taxesAndFees": [{ "included": true }]
              },
              "cancellationPolicies": {
                "refundableTag": "RFN",
                "cancelPolicyInfos": [{ "cancelTime": "YYYY-MM-DD HH:mm:ss" }]
              }
            }
          ]
        }
      ]
    }
  ],
  "guestLevel": 0,
  "sandbox": true
}
```

Display price from `retailRate.total[0]`. Taxes badge from `taxesAndFees[].included`. Refundable vs not from `refundableTag` (`RFN` / `NRFN`).

- `offerId` is what you send to prebook.
- `name` is the room rate label.
- `mappedRoomId` groups rates that are the same room.
- `boardName` is the meal plan.

On the hotel page, call this again with that `hotelId` and **omit** `maxRatesPerHotel` so every rate is returned, then group by `mappedRoomId`.

## 2b. AI search

Same endpoint. Send `aiSearch` instead of `placeId` or `hotelIds`, and set `includeHotelData: true`.

```json
{
  "occupancies": [{ "adults": 2 }],
  "currency": "USD",
  "guestNationality": "US",
  "checkin": "YYYY-MM-DD",
  "checkout": "YYYY-MM-DD",
  "maxRatesPerHotel": 1,
  "roomMapping": true,
  "aiSearch": "romantic getaway in paris",
  "includeHotelData": true
}
```

The response adds a `hotels` array. Render the list from `hotels` (name, `main_photo`, address, rating, `tags`, `story`) and attach prices from `data[]` by matching `hotelId` to `hotels[].id`.

```json
{
  "data": [
    {
      "hotelId": "<id>",
      "roomTypes": [
        {
          "offerId": "<offerId>",
          "rates": [
            {
              "name": "<room>",
              "mappedRoomId": 123,
              "boardName": "Room Only",
              "retailRate": { "total": [{ "amount": 412.76, "currency": "USD" }] },
              "cancellationPolicies": { "refundableTag": "NRFN" }
            }
          ]
        }
      ]
    }
  ],
  "hotels": [
    {
      "id": "lp19fec",
      "name": "Hotel Sainte-Beuve",
      "main_photo": "https://…",
      "address": "9 Rue Sainte-Beuve",
      "rating": 8.75,
      "tags": ["historic building", "wrought iron balconies"],
      "persona": "Cultured Explorer",
      "style": "Classic Parisian Elegance",
      "location_type": "Urban Boutique Hotel",
      "story": "A charming Parisian hotel on a quiet street"
    }
  ],
  "sandbox": true
}
```

## 3. Hotel details

`GET /data/hotel?hotelId={hotelId}&timeout=4`

```json
{
  "data": {
    "id": "<id>",
    "name": "<name>",
    "hotelDescription": "<p>…</p>",
    "hotelImages": [{ "url": "https://...", "defaultImage": true }],
    "hotelImportantInformation": "<p>...</p>",
    "main_photo": "https://path/to/img.png",
    "videoUrl": "https://path/to/optional/video.mp4",
    "city": "<city>",
    "country": "UK",
    "address": "<addr>",
    "hotelFacilities": ["Full-service spa"],
    "starRating": 4,
    "location": { "latitude": 0, "longitude": 0 },
    "rooms": [
      { "id": 123, "roomName": "<room>", "photos": [{ "url": "https://..." }] }
    ],
    "policies": [{ "name": "Pets", "description": "…" }],
    "sentiment_analysis": { "pros": ["…"], "cons": ["…"] }
  }
}
```

`hotelDescription` and `hotelImportantInformation` are HTML. Render them as HTML only after sanitizing, or strip tags and render as text.

Match `rooms[].id` to `mappedRoomId` from the rates response when showing the room photo next to a rate.

## 4. Prebook

`POST https://book.liteapi.travel/v3.0/rates/prebook`

Takes 5–10 seconds. On failure, return the user to the hotel page with an error so they can pick a different rate.

Both fields are required.

```json
{
  "usePaymentSdk": true,
  "offerId": "<offerId from roomTypes>"
}
```

Success (`200`):

```json
{
  "data": {
    "prebookId": "<id>",
    "offerId": "<id>",
    "hotelId": "<id>",
    "price": 690.21,
    "commission": 103.74,
    "currency": "USD",
    "transactionId": "<id>",
    "secretKey": "<key>",
    "paymentTypes": ["NUITEE_PAY"],
    "roomTypes": [
      {
        "rates": [
          {
            "rateId": "<id>",
            "retailRate": {
              "total": [{ "amount": 690.21, "currency": "USD" }],
              "taxesAndFees": [{ "included": true, "amount": 43.61 }]
            },
            "cancellationPolicies": {
              "refundableTag": "RFN",
              "cancelPolicyInfos": [{ "cancelTime": "YYYY-MM-DD HH:mm:ss" }]
            }
          }
        ]
      }
    ]
  }
}
```

Store `prebookId` for the book call. Store `transactionId` and `secretKey` for payment. Show the confirmed price and cancellation policy from `roomTypes[].rates[]` — it can differ from the search price.

Each prebook returns a new `transactionId` and `prebookId`. Persist the pair you sent to the Payment SDK. You cannot look them up later.

Error:

```json
{
  "error": {
    "code": 4002,
    "description": "invalid offerId",
    "message": "required request field is missing or wrong input"
  }
}
```

## 5. Payment SDK

Collect guest details (first name, last name, email) in the UI before mounting the payment form.

Sandbox copy: tell the user to use card `4242424242424242`, any three-digit CVC, and any future expiry.

```html
<script src="https://payment-wrapper.liteapi.travel/dist/liteAPIPayment.js?v=a1"></script>
```

```javascript
var liteAPIConfig = {
  publicKey: "sandbox", // "live" in production. Must match the API key prefix.
  secretKey: "<from prebook response>",
  returnUrl: "<your-booking-page-url>?prebookId=<prebookId>&transactionId=<transactionId>",
  targetElement: "#targetElement",
  appearance: { theme: "flat" },
  options: { business: { name: "Your Business" } },
};
var liteAPIPayment = new LiteAPIPayment(liteAPIConfig);
document.addEventListener("DOMContentLoaded", function () {
  liteAPIPayment.handlePayment();
});
```

`secretKey` comes from prebook. `returnUrl` is your page, where you will call book. The SDK uses Stripe Elements. Style `.lp-submit-button` so the pay button matches the rest of the UI.

Only call book after the SDK redirects to `returnUrl`.

`secretKey` is a payment secret. Keep it on the server when you can, and do not log it. The SDK example above runs in the browser because that is how LiteAPI Payment is loaded; the API key itself still stays on the server.

## 6. Book

`POST https://book.liteapi.travel/v3.0/rates/book`

Takes 5–10 seconds. Success is HTTP 200.

```json
{
  "prebookId": "<from prebook>",
  "holder": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com"
  },
  "payment": {
    "method": "TRANSACTION_ID",
    "transactionId": "<from prebook>"
  },
  "guests": [
    {
      "occupancyNumber": 1,
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com"
    }
  ]
}
```

There is one guest object per room (`occupancyNumber` starting at 1).

```json
{
  "data": {
    "bookingId": "<id>",
    "status": "CONFIRMED",
    "hotelConfirmationCode": "<code>",
    "checkin": "YYYY-MM-DD",
    "checkout": "YYYY-MM-DD",
    "hotel": { "hotelId": "<id>", "name": "<name>" },
    "price": 100,
    "currency": "USD",
    "cancellationPolicies": {
      "refundableTag": "RFN",
      "cancelPolicyInfos": [{ "cancelTime": "YYYY-MM-DD HH:mm:ss" }]
    }
  }
}
```

Show `bookingId` and `hotelConfirmationCode` on the confirmation page.

```json
{
  "error": {
    "code": 4002,
    "description": "missing or empty transaction id",
    "message": "required request field is missing or wrong input"
  }
}
```

Other payment methods exist (`CREDIT`, `WALLET`, account card, card-on-file). Do not switch to them unless the user asks. If you do, read the book endpoint docs first. Card numbers never go to `api.liteapi.travel` or `book.liteapi.travel`; direct card payment uses `https://pci-book.liteapi.travel`.

## 7. Manage bookings

All on `https://book.liteapi.travel/v3.0`.

| Call | Purpose |
|---|---|
| `GET /bookings?guestId=` or `?clientReference=` | List. Optional `customTags=KEY:VALUE,KEY2:VALUE2` (AND across keys). |
| `GET /bookings/{bookingId}` | One booking. |
| `PUT /bookings/{bookingId}` | Cancel. |

`clientReference` is your own id. Send it on book if you want to list by it later.

## Wiring summary

- Places: `GET /data/places` → store `{ placeId, displayName }`.
- Rates: `POST /hotels/rates` with `placeId` or `hotelIds`.
- AI search: `POST /hotels/rates` with `aiSearch` → cards from `hotels[]`, prices from `data[]`.
- Hotel page: `GET /data/hotel`, then rates again for that `hotelId` without `maxRatesPerHotel`, grouped by `mappedRoomId`.
- Prebook: `POST /rates/prebook` with `offerId` → store `prebookId`, `transactionId`, `secretKey`.
- Payment: SDK with `secretKey`, `publicKey` matching the key env, `returnUrl` carrying both ids.
- Book: `POST /rates/book` with `prebookId`, `payment.transactionId`, holder, and guests.
