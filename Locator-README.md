# UPS Locator API – Find Drop-Off & Access Point Locations  
Comprehensive README for Node.js + TypeScript Integrators

---

## 1. What Is the Locator API?
The **UPS Locator API** lets you search for UPS‐branded locations (The UPS Store, Customer Centers, Drop Boxes, Access Points, etc.) near a given address or latitude/longitude. Typical use-cases in shipping apps:

• Provide a "Pick-up at Access Point" option during checkout  
• Offer alternative drop-off points when residential delivery is expensive  
• Show Saturday‐pickup locations to warehouse staff  
• Build "Find a UPS location" store-locator widgets

The API is exposed as a RESTful JSON endpoint secured by OAuth-2 **Client Credentials**.  
Latest version is `v3` (previous v1/v2 are deprecated).

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/locations/{version}/search/availabilities/{reqOption}` | Main search (v3 – use this) |
| `POST` | `/locations/{deprecatedVersion}/search/availabilities/{reqOption}` | Legacy v1 (avoid) |

### 2.1 Path Params
* `version` — `v3` (string, **required**)  
* `deprecatedVersion` — `v1` (string, legacy)  
* `reqOption` — determines WHAT you're searching for (string, **required**)  

```
01 – UPS Locations (Drop-off & Will-Call)
08 – Additional Services list only
16 – Program Types list only
24 – Additional Services + Program Types
32 – Retail Locations list only
40 – Retail Locations + Additional Services
48 – Retail Locations + Program Types
56 – Retail Locations + Additional Services + Program Types
64 – Access Point Locations
```

### 2.2 Query Params
| Name | Default | Notes |
|------|---------|-------|
| `Locale` | `en_US` | 2-char language + 2-char country (`fr_CA`, `es_MX`, …) |

### 2.3 Required Headers
| Header | Description |
|--------|-------------|
| `Authorization: Bearer <token>` | OAuth 2 access token |
| `transId` | 32-char UUID/v4 recommended – used for UPS logging |
| `transactionSrc` | Free-form string identifying your app (max 512) |

---

## 3. TypeScript Interfaces (minimal but complete)

Save as `types/ups-locator.ts`:

```ts
// ---- Auth ----
export interface OAuthToken {
  access_token: string;
  token_type: 'Bearer';
  expires_in: number;          // seconds
}

// ---- Request ----
export interface AddressKeyFormat {
  AddressLine: string;         // "123 Main St"
  PoliticalDivision2: string;  // City
  PoliticalDivision1: string;  // State / Province
  PostcodePrimaryLow: string;  // ZIP / Postal
  CountryCode: string;         // 2-char ISO
  AddressLine2?: string;
  AddressLine3?: string;
  PostcodeExtendedLow?: string; // ZIP+4
}

export interface OriginAddress {
  AddressKeyFormat: AddressKeyFormat;
  MaximumListSize?: string;    // "1"–"50" (default 10)
}

export interface UnitOfMeasurement { Code: 'MI' | 'KM' }

export interface SearchOption {
  OptionType: { Code: '01' | '02' | '03' | '04' | '05' | '06' };
  OptionCode: { Code: string }[]; // e.g. {Code:'001'}
  Relation?: { Code: '01' | '02' }; // 01=AND, 02=OR (only for type 03/04)
}

export interface LocationSearchCriteria {
  SearchOption?: SearchOption[];
  MaximumListSize?: string; // 1-50 (default 5)
  SearchRadius?: string;    // miles/km depending on UOM
  ServiceSearch?: {
    Time?: string;          // "1030"
    ServiceCode?: { Code: '01'|'02'|'03'|'04'|'05' }[];
  };
}

export interface LocatorRequest {
  Request: { RequestAction: 'Locator'; TransactionReference?: { CustomerContext?: string } };
  OriginAddress: OriginAddress;
  Translate: { Locale?: string };                // default en_US
  UnitOfMeasurement: UnitOfMeasurement;          // required for reqOption 01
  LocationSearchCriteria?: LocationSearchCriteria;
}

export interface LocatorRequestWrapper { LocatorRequest: LocatorRequest }

// ---- Response (trimmed) ----
export interface LocatorResponseStatus {
  ResponseStatusCode: '0' | '1';
  ResponseStatusDescription: string;
  Error?: { ErrorSeverity: string; ErrorCode: string; ErrorDescription?: string };
}

export interface DropLocation {
  City: string;
  StateProvince: string;
  CountryCode: string;
  PostalCode: string;
  Distance?: string;           // in MI/KM per UnitOfMeasurement
  LocationID: string;
  LocationType: string;        // "UPS Access Point", "The UPS Store", etc.
  AddressLine?: string[];
  PhoneNumber?: string;
  SaturdayPickupIndicator?: string;
  // …many more fields available – add as needed
}

export interface LocatorResponse {
  Response: LocatorResponseStatus;
  SearchResults?: {
    DropLocation?: DropLocation[];
    Disclaimer?: string;
  };
  AllowAllConfidenceLevels?: 'True' | 'False';
}

export interface LocatorResponseWrapper { LocatorResponse: LocatorResponse }

// ---- Error envelope (for 4xx/5xx HTTP) ----
export interface UpsErrorMessage { code: string; message: string }
export interface UpsErrorResponse { response: { errors: UpsErrorMessage[] } }
```

The schema is enormous; the above covers **90 % of day-to-day use**. Extend as needed.

---

## 4. Authentication (OAuth-2 Client Credentials)

Same process used by every UPS REST API:

```txt
POST https://wwwcie.ups.com/security/v1/oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(<client_id>:<client_secret>)

grant_type=client_credentials
```

• **CIE (test)** base URL: `https://wwwcie.ups.com`  
• **Production** base URL: `https://onlinetools.ups.com`  
Token lifetime ≈ 1 hour – cache and reuse until 5 min before expiry.

---

## 5. Production-Ready Node.js Wrapper

### 5.1 Install Dependencies

```bash
npm i axios qs uuid
npm i -D @types/node
```

### 5.2 `upsLocatorClient.ts`

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import qs from 'qs';
import { v4 as uuid } from 'uuid';
import {
  OAuthToken,
  LocatorRequestWrapper,
  LocatorResponseWrapper,
  UpsErrorResponse
} from './types/ups-locator';

interface UpsLocatorOptions {
  clientId: string;
  clientSecret: string;
  baseURL?: string;                 // defaults to CIE
}

export class UpsLocatorClient {
  private http: AxiosInstance;
  private token?: OAuthToken;
  private tokenFetchedAt = 0;

  constructor(private readonly opts: UpsLocatorOptions) {
    this.http = axios.create({
      baseURL: opts.baseURL ?? 'https://wwwcie.ups.com/api',
      timeout: 15_000
    });
  }

  /* ---------------- Authentication ---------------- */
  private async getToken(): Promise<string> {
    const now = Date.now();
    if (this.token && now - this.tokenFetchedAt < (this.token.expires_in - 60) * 1_000) {
      return this.token.access_token;
    }

    const auth = Buffer.from(`${this.opts.clientId}:${this.opts.clientSecret}`).toString('base64');
    const { data } = await axios.post<OAuthToken>(
      `${this.http.defaults.baseURL?.replace('/api', '')}/security/v1/oauth/token`,
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

  /* ---------------- API Call ---------------- */
  /**
   * Search UPS locations (drop-off, access points, etc.)
   * @param body LocatorRequest wrapper
   * @param reqOption see README (01, 32, 64 …)
   * @param locale   "en_US" etc.
   */
  async searchLocations(
    body: LocatorRequestWrapper,
    reqOption: '01' | '08' | '16' | '24' | '32' | '40' | '48' | '56' | '64' = '01',
    locale = 'en_US'
  ): Promise<LocatorResponseWrapper> {
    const token = await this.getToken();

    const { data } = await this.http.post<LocatorResponseWrapper>(
      `/locations/v3/search/availabilities/${reqOption}`,
      body,
      {
        params: { Locale: locale },
        headers: {
          Authorization: `Bearer ${token}`,
          transId: uuid().replace(/-/g, ''),
          transactionSrc: 'my-shipping-app'
        }
      }
    );

    return data;
  }
}

/* ------------- Helper for UPS error payload -------------- */
export function handleUpsError(err: unknown): never {
  if (axios.isAxiosError(err)) {
    const e = err as AxiosError<UpsErrorResponse>;
    console.error('UPS API error:', e.response?.data?.response?.errors ?? e.message);
  } else {
    console.error('Unknown error', err);
  }
  throw err; // re-throw for upstream handler
}
```

### 5.3 Using the Client

```ts
import { UpsLocatorClient } from './upsLocatorClient';
import { LocatorRequestWrapper } from './types/ups-locator';

(async () => {
  const ups = new UpsLocatorClient({
    clientId: process.env.UPS_CLIENT_ID!,
    clientSecret: process.env.UPS_CLIENT_SECRET!,
    baseURL: process.env.NODE_ENV === 'production'
      ? 'https://onlinetools.ups.com/api'
      : 'https://wwwcie.ups.com/api'
  });

  const request: LocatorRequestWrapper = {
    LocatorRequest: {
      Request: { RequestAction: 'Locator' },
      OriginAddress: {
        AddressKeyFormat: {
          AddressLine: '26601 ALISO CREEK RD',
          PoliticalDivision2: 'ALISO VIEJO',
          PoliticalDivision1: 'CA',
          PostcodePrimaryLow: '92656',
          CountryCode: 'US'
        }
      },
      Translate: { Locale: 'en_US' },
      UnitOfMeasurement: { Code: 'MI' },
      LocationSearchCriteria: { SearchRadius: '25' } // 25 miles
    }
  };

  try {
    const resp = await ups.searchLocations(request, '64'); // 64 = Access Points
    const results = resp.LocatorResponse.SearchResults?.DropLocation ?? [];

    console.log(`Found ${results.length} location(s)`);
    results.forEach(l =>
      console.log(`${l.LocationID} – ${l.LocationType} – ${l.Distance} mi – ${l.City}`));
  } catch (err) {
    handleUpsError(err);
  }
})();
```

---

## 6. Error Handling Patterns
UPS uses standard HTTP codes **plus** a JSON body:

* `400` – validation failure (missing/invalid fields)  
* `401` – bad/expired token  
* `403` – blocked merchant / permission issue  
* `429` – rate-limit exceeded (apply exponential back-off)  
* `5xx` – UPS internal error (retry with jitter)

Always wrap calls in `try/catch`, inspect `error.response.status`, and surface user-friendly messages ("UPS service temporarily unavailable, please retry").

---

## 7. Best Practices for Locator Searches

1. **Cache tokens** centrally (in-memory or Redis) to avoid hitting `/oauth/token` each request.  
2. **Throttle** UI searches (debounce 500 ms) to respect UPS rate limits.  
3. For **checkout** flows, pre-fetch locations server-side once the user enters ZIP + country, then lazy-load on the client.  
4. Use **`reqOption 64`** to return only Access Points when offering "hold at Access Point".  
5. Always pass a **reasonable `SearchRadius`** (e.g. 25 mi) to keep responses snappy and avoid long JSON payloads.  
6. Display the **Disclaimer** text if returned – UPS requires it when the results exceed the requested service mix.  
7. Log `transId` + `TransactionReference.CustomerContext` to trace specific UPS calls in production.  
8. Respect UPS brand usage: show official location names and IDs exactly as returned.  
9. Combine with **Address Validation** first; a clean, geocodable address yields better Locator accuracy.  
10. When you must support multiple carriers, normalize the Locator response into a carrier-agnostic shape in your domain layer.

---

Happy coding & happy shipping!
