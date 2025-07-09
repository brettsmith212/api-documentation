# UPS Rating API  
Comprehensive README for Node.js + TypeScript Integrators  

---

## 1. What Is the Rating API?
The **Rating API** lets you obtain UPS shipping prices (and optional transit-times) for a single shipment ("_Rate_") or to shop all available services ("_Shop_").  
Typical use-cases in shipping platforms:

* Show live shipping options and costs in checkout
* Select the cheapest / fastest service in back-office rules
* Quote negotiated contract rates for your own Shipper Number
* Pre-calculate duties/taxes (with Landed-Cost) after choosing a service

Like all modern UPS APIs, Rating is REST + JSON, secured with OAuth-2 *Client-Credentials*.  
Current stable path version is `v2409`.

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/rating/{version}/{requestoption}` | Get rates / shop / include transit-times |

### 2.1 Required Headers
| Name | Notes |
|------|-------|
| `transId` | Your own **32-char** correlation ID (UUID or ULID) |
| `transactionSrc` | Free-form app identifier (≤512 chars). Default `testing` |

### 2.2 Path Params
| Param | Type | Example | Description |
|-------|------|---------|-------------|
| `version` | `'v2409'` (string) | `v2409` | Indicates response schema version. |
| `requestoption` | enum | `Rate` | What the server should return (see below). |

`requestoption` values:  
• **Rate** (default) – one service you specify in body.  
• **Shop** – return rates for *all* services.  
• **Ratetimeintransit** – rate one service **+** include time-in-transit.  
• **Shoptimeintransit** – shop all services **+** transit-times.

### 2.3 Query Params
| Name | Type | Notes |
|------|------|-------|
| `additionalinfo` | `"timeintransit"` | *Only* when you also pass `rate` in path. |

### 2.4 Request Body (JSON ‑ minimal example)
Wrapper object is `RateRequest`:

```jsonc
{
  "RateRequest": {
    "Request": { "RequestOption": "Rate" },
    "Shipment": {
      "Shipper": {
        "Name": "ACME Corp",
        "ShipperNumber": "A1B2C3",
        "Address": {
          "AddressLine": ["123 Main St"],
          "City": "Timonium",
          "StateProvinceCode": "MD",
          "PostalCode": "21093",
          "CountryCode": "US"
        }
      },
      "ShipTo": {
        "Name": "John Doe",
        "Address": {
          "AddressLine": ["500 Tech Way"],
          "City": "Alpharetta",
          "StateProvinceCode": "GA",
          "PostalCode": "30005",
          "CountryCode": "US",
          "ResidentialAddressIndicator": "Y"
        }
      },
      "ShipFrom": { /* similar to Shipper */ },
      "Service": { "Code": "03" },            // Ground
      "PaymentDetails": {
        "ShipmentCharge": [{
          "Type": "01",                       // Transportation
          "BillShipper": { "AccountNumber": "A1B2C3" }
        }]
      },
      "Package": {
        "PackagingType": { "Code": "02" },    // Package/customer supplied
        "Dimensions": {
          "UnitOfMeasurement": { "Code": "IN" },
          "Length": "5", "Width": "5", "Height": "5"
        },
        "PackageWeight": {
          "UnitOfMeasurement": { "Code": "LBS" },
          "Weight": "1"
        }
      },
      "NumOfPieces": "1"
    }
  }
}
```

---

## 3. TypeScript Interfaces (minimal but useful)

```ts
// ----- Shared OAuth -----
export interface OAuthToken {
  access_token: string;
  token_type: 'Bearer';
  expires_in: number;            // seconds
}

// ----- Rating Request -----
export interface Address {
  AddressLine: string[];
  City: string;
  StateProvinceCode: string;
  PostalCode: string;
  CountryCode: string;
  ResidentialAddressIndicator?: 'Y';
}

export interface Party {
  Name: string;
  ShipperNumber?: string;        // only for Shipper
  Address: Address;
}

export interface PackageDimensions {
  UnitOfMeasurement: { Code: 'IN' | 'CM' };
  Length: string; Width: string; Height: string;
}

export interface PackageWeight {
  UnitOfMeasurement: { Code: 'LBS' | 'KGS' };
  Weight: string;
}

export interface Package {
  PackagingType: { Code: '02' | '01' | string };
  Dimensions: PackageDimensions;
  PackageWeight: PackageWeight;
  SimpleRate?: { Code: 'XS' | 'S' | 'M' | 'L' | 'XL'; Description?: string };
  OversizeIndicator?: 'X';
  MinimumBillableWeightIndicator?: 'X';
}

export interface RateRequest {
  Request: { RequestOption: 'Rate' | 'Shop' | 'Ratetimeintransit' | 'Shoptimeintransit'; SubVersion?: string };
  PickupType?: { Code: '01' | '03' | '06' | '11' | string };
  CustomerClassification?: { Code: string };
  Shipment: {
    Shipper: Party;
    ShipTo: Party;
    ShipFrom: Party;
    Service?: { Code: string; Description?: string }; // required for Rate/RateTimeInTransit
    PaymentDetails: {
      ShipmentCharge: Array<
        | { Type: '01'; BillShipper: { AccountNumber: string } }
        | { Type: '01'; BillThirdParty: { AccountNumber: string; Address: Address } }
      >;
    };
    Package: Package | Package[];   // array for multi-piece
    NumOfPieces?: string;
    ShipmentRatingOptions?: { NegotiatedRatesIndicator?: 'Y' };
    ShipmentTotalWeight?: PackageWeight;
  };
}

export interface RateRequestWrapper { RateRequest: RateRequest; }

// ----- Rating Response (trimmed) -----
export interface MonetaryValue { CurrencyCode: string; MonetaryValue: string; }

export interface RatedShipment {
  Service: { Code: string; Description: string };
  GuaranteedDaysToDelivery?: string;
  RatedShipmentAlert?: { Code: string; Description: string }[];
  TotalCharges: MonetaryValue;
  NegotiatedRateCharges?: MonetaryValue;
  TimeInTransit?: { ServiceSummary: any }; // simplify
}

export interface RateResponse {
  Response: { ResponseStatus: { Code: '1' | '0'; Description: string }; Alert?: any[] };
  RatedShipment: RatedShipment[];
}

export interface RateResponseWrapper { RateResponse: RateResponse; }

// ----- UPS Error wrapper -----
export interface UpsError { code: string; message: string; }
export interface UpsErrorResponse { response: { errors: UpsError[] } }
```

Save as `types/ups-rating.ts` to get full IntelliSense.

---

## 4. Authentication (OAuth-2 Client-Credentials)

```txt
POST https://wwwcie.ups.com/security/v1/oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(<client_id>:<client_secret>)

grant_type=client_credentials
```

• CIE (test) base URL: `https://wwwcie.ups.com`  
• Production: `https://onlinetools.ups.com`  

Cache the token (~1 h lifetime) and reuse until 60 s before expiry.

---

## 5. Production-Ready Node.js Example with Axios

### 5.1 Install

```bash
npm i axios qs
npm i -D @types/node
```

### 5.2 `upsRateClient.ts`

```ts
import axios, { AxiosError, AxiosInstance } from 'axios';
import qs from 'qs';
import {
  OAuthToken,
  RateRequestWrapper,
  RateResponseWrapper,
  UpsErrorResponse
} from './types/ups-rating';

interface UpsRateClientOpts {
  clientId: string;
  clientSecret: string;
  baseURL?: string;          // defaults to CIE
}

export class UpsRateClient {
  private http: AxiosInstance;
  private token?: OAuthToken;
  private tokenFetchedAt = 0;

  constructor(private readonly opts: UpsRateClientOpts) {
    this.http = axios.create({
      baseURL: opts.baseURL ?? 'https://wwwcie.ups.com/api',
      timeout: 15_000
    });
  }

  /* ---------- Auth ---------- */
  private async getToken(): Promise<string> {
    const now = Date.now();
    if (this.token && now - this.tokenFetchedAt < (this.token.expires_in - 60) * 1000) {
      return this.token.access_token;
    }
    const auth = Buffer.from(`${this.opts.clientId}:${this.opts.clientSecret}`).toString('base64');
    const { data } = await axios.post<OAuthToken>(
      `${this.http.defaults.baseURL!.replace('/api', '')}/security/v1/oauth/token`,
      qs.stringify({ grant_type: 'client_credentials' }),
      {
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
          Authorization: `Basic ${auth}`
        }
      }
    );
    this.token = data;
    this.tokenFetchedAt = now;
    return data.access_token;
  }

  /* ---------- Main call ---------- */
  async rate(
    body: RateRequestWrapper,
    pathRequestOption: 'Rate' | 'Shop' | 'Ratetimeintransit' | 'Shoptimeintransit' = 'Rate',
    query?: { additionalinfo?: 'timeintransit' },
    headers?: { transId?: string; transactionSrc?: string }
  ): Promise<RateResponseWrapper> {
    const token = await this.getToken();

    const { data } = await this.http.post<RateResponseWrapper>(
      `/rating/v2409/${pathRequestOption}`,
      body,
      {
        params: query,
        headers: {
          Authorization: `Bearer ${token}`,
          transId: headers?.transId ?? crypto.randomUUID(),
          transactionSrc: headers?.transactionSrc ?? 'my-shipping-app'
        }
      }
    );
    return data;
  }
}

/* ---------- Helper ---------- */
export function handleUpsError(err: unknown) {
  if (axios.isAxiosError(err)) {
    const e = err as AxiosError<UpsErrorResponse>;
    console.error('UPS error:', e.response?.data?.response?.errors ?? e.message);
  } else {
    console.error('Unknown error', err);
  }
}
```

### 5.3 Usage Samples

**a) Rate one service (Ground):**

```ts
import { UpsRateClient } from './upsRateClient';
import { RateRequestWrapper } from './types/ups-rating';

const ups = new UpsRateClient({
  clientId: process.env.UPS_CLIENT_ID!,
  clientSecret: process.env.UPS_CLIENT_SECRET!,
  baseURL: process.env.NODE_ENV === 'production'
    ? 'https://onlinetools.ups.com/api'
    : 'https://wwwcie.ups.com/api'
});

const body: RateRequestWrapper = { /*… see JSON above …*/ };

(async () => {
  try {
    const { RateResponse } = await ups.rate(body, 'Rate');
    const first = RateResponse.RatedShipment[0];
    console.log(`Ground price: ${first.TotalCharges.MonetaryValue} ${first.TotalCharges.CurrencyCode}`);
  } catch (err) {
    handleUpsError(err);
  }
})();
```

**b) Shop all services + transit times:**

```ts
const body: RateRequestWrapper = {
  RateRequest: { ...original.RateRequest, Request: { RequestOption: 'Shop' } }
};

const { RateResponse } = await ups.rate(
  body,
  'Shoptimeintransit',
  { additionalinfo: 'timeintransit' }
);

RateResponse.RatedShipment.forEach(s =>
  console.log(`${s.Service.Description}: ${s.TotalCharges.MonetaryValue} ETA=${s.GuaranteedDaysToDelivery}`));
```

---

## 6. Error-Handling Patterns

UPS returns **both** HTTP status + JSON error list.

Common statuses:

* `400` – validation error (missing fields, wrong codes)
* `401` – bad/expired token
* `403` – account blocked / lacking permission
* `429` – rate-limited (back-off)
* `500` – UPS internal error

Recommended wrapper (see `handleUpsError`) to surface `error.response.data.response.errors`.

Implement exponential retry only for `429` and `>=500`.

---

## 7. Best Practices for Accurate Rate Calculations

1. **Always pass dimensions & weight** – dimensional weight often overrides actual weight.  
2. **Inform Residential vs Commercial** (`ResidentialAddressIndicator`) to avoid unexpected surcharges.  
3. **Use Negotiated Rates** – set `ShipmentRatingOptions.NegotiatedRatesIndicator = "Y"` when you have a contract.  
4. **Cache results** during checkout session; rates rarely change within minutes.  
5. **Shop then filter** – call `Shop` once, cache list, and let UI or rules choose cheapest/fastest.  
6. **Validate addresses first** (Address Validation API) to avoid mis-rated rural/residential misclassification.  
7. **Respect UPS rate-limits** – bursts of >25 RPS may be throttled; queue or sleep.  
8. **Send `transId` correlation IDs** – will appear in UPS logs for support tickets.  
9. **Update when new version drops** – e.g., `v2409` always returns `RatedShipment` as array, simplifying parsing.  
10. **Sandbox vs Production** – sandbox prices are *not* contract accurate; only check logic, not dollars.

---

Happy rating & shipping!
