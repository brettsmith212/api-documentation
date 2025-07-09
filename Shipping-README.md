# UPS Shipping API  
Comprehensive README for Node.js + TypeScript Integrators  

---

## 1. What Is the Shipping API?

The **Shipping API** (often called "Ship API") is the work-horse that:

* Generates **official UPS labels** (GIF/PNG/ZPL, incl. return labels)  
* Returns **tracking numbers** & **shipment charges** in real-time  
* Allows you to **void / cancel** labels that you no longer need  

It is a REST/JSON service secured with OAuth 2 **Client-Credential** tokens.  
Current stable version path is `v2409`.

Typical use-cases:

* E-commerce checkout or WMS printing outbound labels  
* Marketplace platforms issuing labels on behalf of sellers  
* OMS / returns-portal generating pre-paid return labels  
* Back-office batched label creation and later voiding mis-prints  

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/shipments/{version}/ship` | Create a shipment & label |
| `DELETE` | `/shipments/{version}/void/cancel/{shipmentIdentificationNumber}` | Void (cancel) a previously created label |
| `POST` | `/shipments/{deprecatedVersion}/ship` | Legacy SOAP-style wrapper (avoid) |
| `DELETE` | `/shipments/{deprecatedVersion}/void/cancel/{shipmentIdentificationNumber}` | Legacy void (avoid) |

### 2.1 Required Headers

| Name | Notes |
|------|-------|
| `transId` | Your own **32-char** correlation ID (UUID/ULID) |
| `transactionSrc` | Free-form identifier for your app (≤ 512 chars). Default `testing` |

### 2.2 Path Params

| Param | Type | Example | Description |
|-------|------|---------|-------------|
| `version` | `'v2409'` | `v2409` | Response schema/features version |
| `shipmentIdentificationNumber` (void) | string | `1Z999AA10123456784` | Master "1Z…" number to void |

### 2.3 Query Params

| Name | Type | Notes |
|------|------|-------|
| `additionaladdressvalidation` | `"city"` | If present UPS also validates the **city** of Ship-To |

### 2.4 Request Body (JSON – minimal working example)

Wrapper is **`ShipmentRequest`**:

```jsonc
{
  "ShipmentRequest": {
    "Request": {             // meta
      "RequestOption": "nonvalidate",   // or "validate"
      "SubVersion": "2409"
    },
    "Shipment": {
      "Description": "Website order #1234",
      "Shipper": {
        "Name": "ACME Corp",
        "ShipperNumber": "A1B2C3",
        "Phone": { "Number": "5555550000" },
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
        "Phone": { "Number": "5555551111" },
        "Address": {
          "AddressLine": ["500 Tech Way"],
          "City": "Alpharetta",
          "StateProvinceCode": "GA",
          "PostalCode": "30005",
          "CountryCode": "US",
          "Residential": "Y"
        }
      },
      "Service": { "Code": "03" },                // Ground
      "PaymentInformation": {
        "ShipmentCharge": {
          "Type": "01",                           // Transportation
          "BillShipper": { "AccountNumber": "A1B2C3" }
        }
      },
      "Package": {
        "Packaging": { "Code": "02" },            // Customer supplied
        "Dimensions": {
          "UnitOfMeasurement": { "Code": "IN" },
          "Length": "10", "Width": "6", "Height": "4"
        },
        "PackageWeight": {
          "UnitOfMeasurement": { "Code": "LBS" },
          "Weight": "2"
        }
      }
    },
    "LabelSpecification": {
      "LabelImageFormat": { "Code": "GIF" },      // ZPL, PNG, EPL, PDF also valid
      "HTTPUserAgent": "my-shipping-service/1.0"
    }
  }
}
```

---

## 3. TypeScript Interfaces (concise but useful)

```ts
// ---------- OAuth ----------
export interface OAuthToken {
  access_token: string;
  token_type: 'Bearer';
  expires_in: number; // seconds
}

// ---------- Address ----------
export interface Address {
  AddressLine: string[];
  City: string;
  StateProvinceCode: string;
  PostalCode: string;
  CountryCode: string;
  Residential?: 'Y';
}

// ---------- Parties ----------
export interface Party {
  Name: string;
  AttentionName?: string;
  ShipperNumber?: string;      // only for Shipper
  Phone?: { Number: string; Extension?: string };
  FaxNumber?: string;
  TaxIdentificationNumber?: string;
  Address: Address;
}

// ---------- Packages ----------
export interface PackageDimensions {
  UnitOfMeasurement: { Code: 'IN' | 'CM' };
  Length: string; Width: string; Height: string;
}
export interface PackageWeight {
  UnitOfMeasurement: { Code: 'LBS' | 'KGS' };
  Weight: string;
}
export interface Package {
  Description?: string;
  Packaging: { Code: '02' | '01' | string };
  Dimensions?: PackageDimensions;
  PackageWeight: PackageWeight;
}

// ---------- Shipment ----------
export interface Shipment {
  Description?: string;
  Shipper: Party;
  ShipTo: Party;
  ShipFrom?: Party;
  Service: { Code: string; Description?: string };
  Package: Package | Package[];
  PaymentInformation: {
    ShipmentCharge: {
      Type: '01' | '02' | '03'; // 01-Transportation
      BillShipper?: { AccountNumber: string };
      BillThirdParty?: { AccountNumber: string; Address: Address };
      BillReceiver?: { AccountNumber: string };
    };
  };
  ShipmentRatingOptions?: { NegotiatedRatesIndicator?: 'X' };
}

// ---------- Request Wrapper ----------
export interface ShipmentRequest {
  Request: { RequestOption: 'validate' | 'nonvalidate'; SubVersion?: string; TransactionReference?: { CustomerContext?: string } };
  Shipment: Shipment;
  LabelSpecification?: {
    LabelImageFormat: { Code: 'GIF' | 'PNG' | 'ZPL' | 'PDF' };
    HTTPUserAgent?: string;
  };
}
export interface SHIPRequestWrapper { ShipmentRequest: ShipmentRequest; }

// ---------- Response (trimmed) ----------
export interface MonetaryValue { CurrencyCode: string; MonetaryValue: string; }

export interface ShipmentResults {
  ShipmentCharges: {
    TransportationCharges: MonetaryValue;
    ServiceOptionsCharges?: MonetaryValue;
    TotalCharges: MonetaryValue;
  };
  NegotiatedRateCharges?: { TotalCharges: MonetaryValue };
  ShipmentIdentificationNumber: string;     // "1Z…"
  PackageResults: Array<{
    TrackingNumber: string;
    LabelImage: { LabelImageFormat: { Code: string }; GraphicImage: string }; // base64
  }>;
}

export interface ShipmentResponse {
  Response: { ResponseStatus: { Code: '1' | '0'; Description: string }; Alert?: any[] };
  ShipmentResults: ShipmentResults;
}
export interface SHIPResponseWrapper { ShipmentResponse: ShipmentResponse; }

// ---------- UPS Error wrapper ----------
export interface UpsError { code: string; message: string; }
export interface UpsErrorResponse { response: { errors: UpsError[] } }
```

Save as `types/ups-shipping.ts` for full IntelliSense support.

---

## 4. Production-Ready Node.js Implementation (Axios)

### 4.1 Install dependencies

```bash
npm i axios qs
npm i -D @types/node
```

### 4.2 `upsShippingClient.ts`

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import qs from 'qs';
import {
  OAuthToken,
  SHIPRequestWrapper,
  SHIPResponseWrapper,
  UpsErrorResponse
} from './types/ups-shipping';

interface UpsShippingClientOptions {
  clientId: string;
  clientSecret: string;
  baseURL?: string;               // default CIE
}

export class UpsShippingClient {
  private axios: AxiosInstance;
  private token?: OAuthToken;
  private tokenFetchedAt = 0;

  constructor(private readonly opts: UpsShippingClientOptions) {
    this.axios = axios.create({
      baseURL: opts.baseURL ?? 'https://wwwcie.ups.com/api',
      timeout: 20_000
    });
  }

  // -------- Private helpers --------
  private async getToken(): Promise<string> {
    const now = Date.now();
    if (this.token && now - this.tokenFetchedAt < (this.token.expires_in - 60) * 1000) {
      return this.token.access_token;
    }
    const basic = Buffer.from(`${this.opts.clientId}:${this.opts.clientSecret}`).toString('base64');
    const { data } = await axios.post<OAuthToken>(
      `${this.axios.defaults.baseURL?.replace('/api', '')}/security/v1/oauth/token`,
      qs.stringify({ grant_type: 'client_credentials' }),
      {
        headers: { 'Content-Type': 'application/x-www-form-urlencoded', Authorization: `Basic ${basic}` }
      }
    );
    this.token = data;
    this.tokenFetchedAt = now;
    return data.access_token;
  }

  private authHeader(token: string) { return { Authorization: `Bearer ${token}` }; }

  // -------- Public API --------

  /** Create shipment & label */
  async createShipment(
    body: SHIPRequestWrapper,
    params?: { additionaladdressvalidation?: 'city' },
    version: string = 'v2409'
  ): Promise<SHIPResponseWrapper> {
    const token = await this.getToken();
    const { data } = await this.axios.post<SHIPResponseWrapper>(
      `/shipments/${version}/ship`,
      body,
      { headers: this.authHeader(token), params }
    );
    return data;
  }

  /** Void previously created shipment */
  async voidShipment(shipmentId: string, version = 'v2409'): Promise<void> {
    const token = await this.getToken();
    await this.axios.delete(
      `/shipments/${version}/void/cancel/${shipmentId}`,
      { headers: this.authHeader(token) }
    );
  }

  /** Expose raw axios if you need fine-grained control */
  get http() { return this.axios; }
}

// Handy error parser
export function parseUpsError(err: unknown): string {
  if (axios.isAxiosError(err)) {
    const e = err as AxiosError<UpsErrorResponse>;
    return JSON.stringify(e.response?.data?.response?.errors ?? e.message);
  }
  return 'Unknown error';
}
```

### 4.3 Usage Example

```ts
import { UpsShippingClient, parseUpsError } from './upsShippingClient';
import { SHIPRequestWrapper } from './types/ups-shipping';

(async () => {
  const ups = new UpsShippingClient({
    clientId: process.env.UPS_CLIENT_ID!,
    clientSecret: process.env.UPS_CLIENT_SECRET!,
    baseURL: process.env.NODE_ENV === 'production'
      ? 'https://onlinetools.ups.com/api'
      : 'https://wwwcie.ups.com/api'
  });

  const shipReq: SHIPRequestWrapper = { /* ← fill as shown in section 2.4 */ };

  try {
    const resp = await ups.createShipment(shipReq);
    const results = resp.ShipmentResponse.ShipmentResults;
    console.log('Tracking #: ', results.PackageResults[0].TrackingNumber);
    console.log('Label (base64 GIF): ', results.PackageResults[0].LabelImage.GraphicImage);
  } catch (err) {
    console.error('UPS error:', parseUpsError(err));
  }
})();
```

---

## 5. Authentication (OAuth-2 Client-Credentials)

```
POST https://wwwcie.ups.com/security/v1/oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(<client_id>:<client_secret>)

grant_type=client_credentials
```

Token lifetime ≈ 1 hour — **cache & reuse** until 60 s before expiry.  
Use `https://onlinetools.ups.com` in production.

---

## 6. Error-Handling Patterns

UPS combines HTTP codes **and** JSON error bodies (`response.errors[]`).

Common statuses:

* `400` – schema/validation error (`additionalInfo`: field, code)  
* `401` – invalid/expired token  
* `403` – blocked merchant or no shipping permission  
* `429` – rate limited (respect `Retry-After`)  
* `5xx` – UPS internal (retry with exponential back-off)

Always wrap calls in `try/catch`, map UPS codes to domain errors, and implement retries for `429`/`5xx`.

---

## 7. Best Practices for Creating Labels in Production

1. **Validate addresses first** (Address-Validation API) to avoid label failures.  
2. **Cache service codes** and negotiated rate flags per account to reduce payload size.  
3. **Generate transId UUIDs** and log them – UPS uses them for support escalations.  
4. **Store raw `ShipmentRequest` & `ShipmentResponse`** for audit/re-print / customs disputes.  
5. Prefer **ZPL** or **EPL** for thermal printers; use **GIF/PNG/PDF** for laser printing.  
6. For multi-package shipments, send an **array** in `Package` (UPS returns one label per pkg).  
7. **Void unused labels ASAP** (UPS billing may occur if not voided within pickup window).  
8. Respect **version bump** (e.g., `v2412` future) — always keep `{version}` in env variable.  
9. Shipments to Puerto Rico or International: include **customs forms** via `ShipmentServiceOptions.InternationalForms`.  
10. Monitor `Alert` codes in response – they often contain actionable warnings (missing CN22, address fixes, surcharges).  

---

Happy printing & safe shipping!
