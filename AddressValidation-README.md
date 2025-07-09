# UPS Address Validation – Street Level  
Comprehensive README for Node.js + TypeScript Integrators  

---

## 1. What Is the Address Validation API?
The **Address Validation – Street Level API** verifies a U.S. or Puerto-Rico address against the USPS database and (optionally) tells you whether it is residential or commercial. Typical use-cases in shipping apps:

* Prevent shipping-label failures caused by bad addresses  
* Improve carrier rate estimates (residential vs commercial)  
* Auto-correct user-entered addresses or present a candidate list  

UPS exposes the service as a RESTful JSON endpoint secured by OAuth-2 **Client-Credentials**.  
Two versions exist (`v2` = current, `v1` = deprecated).

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/addressvalidation/{version}/{requestOption}` | Main validation & classification |
| `POST` | `/addressvalidation/{deprecatedVersion}/{requestOption}` | Legacy v1 (avoid) |

### 2.1 Path Params
* `version` — currently only `v2` (string, **required**)  
* `requestOption` — what you want the API to do (int, **required**)  
  1 = Validate address only  
  2 = Classify address only  
  3 = Validate **and** classify  

### 2.2 Query Params
| Name | Type | Default | Notes |
|------|------|---------|-------|
| `regionalrequestindicator` | `"True" \| "False"` | `False` | If `True`, UPS validates *region-level* (city/state/ZIP) instead of exact street. |
| `maximumcandidatelistsize` | `0-50` | `15` | How many candidate addresses to return when ambiguous. |

### 2.3 Request Body (JSON)
Top-level wrapper is `XAVRequest` with two key objects:  

```jsonc
{
  "XAVRequest": {
    "Request": { "RequestOption": "1" },
    "AddressKeyFormat": {
      "AddressLine": ["26601 ALISO CREEK ROAD", "STE D"],
      "PoliticalDivision2": "ALISO VIEJO",
      "PoliticalDivision1": "CA",
      "PostcodePrimaryLow": "92656",
      "CountryCode": "US"
    }
  }
}
```

---

## 3. TypeScript Interfaces (minimal but complete)

```ts
// Shared
export interface OAuthToken {
  access_token: string;
  token_type: 'Bearer';
  expires_in: number; // seconds
}

// Request
export interface AddressKeyFormat {
  ConsigneeName?: string;
  AttentionName?: string;
  AddressLine?: string[];
  Region?: string;
  PoliticalDivision2?: string; // city
  PoliticalDivision1?: string; // state
  PostcodePrimaryLow?: string;
  PostcodeExtendedLow?: string;
  Urbanization?: string;
  CountryCode: string; // 2-char ISO
}

export interface XAVRequest {
  Request: {
    RequestOption: '1' | '2' | '3';
    TransactionReference?: { CustomerContext?: string };
  };
  RegionalRequestIndicator?: 'True' | 'False';
  MaximumCandidateListSize?: string; // "0"-"50"
  AddressKeyFormat: AddressKeyFormat;
}

export interface XAVRequestWrapper {
  XAVRequest: XAVRequest;
}

// Response (trimmed to most-used)
export interface ResponseStatus {
  Code: '0' | '1';
  Description: 'Success' | 'Failure';
}

export interface CandidateAddressKeyFormat extends AddressKeyFormat {
  AddressLine?: string[]; // always array in v2
}

export interface Candidate {
  AddressClassification?: { Code: '0' | '1' | '2'; Description: string };
  AddressKeyFormat: CandidateAddressKeyFormat;
}

export interface XAVResponse {
  Response: { ResponseStatus: ResponseStatus };
  ValidAddressIndicator?: string;
  AmbiguousAddressIndicator?: string;
  NoCandidatesIndicator?: string;
  AddressClassification?: { Code: '0' | '1' | '2'; Description: string };
  Candidate?: Candidate[];
}

export interface XAVResponseWrapper {
  XAVResponse: XAVResponse;
}

// Error shape
export interface UpsErrorMessage { code: string; message: string }
export interface UpsErrorResponse { response: { errors: UpsErrorMessage[] } }
```

Save these in `types/ups-address-validation.ts` for IDE auto-completion.

---

## 4. Authentication (OAuth-2 Client-Credentials)

```txt
POST https://wwwcie.ups.com/security/v1/oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(<client_id>:<client_secret>)

grant_type=client_credentials
```

* **CIE (test)** base URL: `https://wwwcie.ups.com`  
* **Production** base URL: `https://onlinetools.ups.com`  

Token lifetime ≈ 1 hour — cache and reuse until expiry.

---

## 5. Production-Ready Node.js Implementation

The example uses `axios` but can be swapped for `fetch` in Node 18+.

### 5.1 Install Deps

```bash
npm i axios qs
npm i -D @types/node
```

### 5.2 `upsClient.ts` – thin wrapper

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import qs from 'qs';
import {
  OAuthToken,
  XAVRequestWrapper,
  XAVResponseWrapper,
  UpsErrorResponse
} from './types/ups-address-validation';

interface UpsClientOptions {
  clientId: string;
  clientSecret: string;
  baseURL?: string; // default CIE
}

export class UpsClient {
  private axios: AxiosInstance;
  private token?: OAuthToken;
  private tokenFetchedAt = 0;

  constructor(private readonly opts: UpsClientOptions) {
    this.axios = axios.create({
      baseURL: opts.baseURL ?? 'https://wwwcie.ups.com/api',
      timeout: 15_000
    });
  }

  /** Get (and cache) access token */
  private async getToken(): Promise<string> {
    const now = Date.now();
    if (this.token && now - this.tokenFetchedAt < (this.token.expires_in - 60) * 1000) {
      return this.token.access_token;
    }

    const auth = Buffer.from(`${this.opts.clientId}:${this.opts.clientSecret}`).toString('base64');
    const { data } = await axios.post<OAuthToken>(
      `${this.axios.defaults.baseURL?.replace('/api', '')}/security/v1/oauth/token`,
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

  /** Perform Address Validation */
  async validateAddress(
    requestBody: XAVRequestWrapper,
    requestOption: 1 | 2 | 3 = 1,
    params?: { regionalrequestindicator?: 'True' | 'False'; maximumcandidatelistsize?: number }
  ): Promise<XAVResponseWrapper> {
    const token = await this.getToken();

    const { data } = await this.axios.post<XAVResponseWrapper>(
      `/addressvalidation/v2/${requestOption}`,
      requestBody,
      {
        params,
        headers: { Authorization: `Bearer ${token}` }
      }
    );

    return data;
  }

  /** Optional: expose raw axios for advanced use */
  get http() { return this.axios; }
}
```

### 5.3 Using the Client

```ts
import { UpsClient } from './upsClient';
import { XAVRequestWrapper } from './types/ups-address-validation';

(async () => {
  const ups = new UpsClient({
    clientId: process.env.UPS_CLIENT_ID!,
    clientSecret: process.env.UPS_CLIENT_SECRET!,
    baseURL: process.env.NODE_ENV === 'production'
      ? 'https://onlinetools.ups.com/api'
      : 'https://wwwcie.ups.com/api'
  });

  const body: XAVRequestWrapper = {
    XAVRequest: {
      Request: { RequestOption: '3' },           // validate + classify
      AddressKeyFormat: {
        AddressLine: ['26601 ALISO CREEK ROAD', 'STE D'],
        PoliticalDivision2: 'ALISO VIEJO',
        PoliticalDivision1: 'CA',
        PostcodePrimaryLow: '92656',
        CountryCode: 'US'
      }
    }
  };

  try {
    const resp = await ups.validateAddress(body, 3);
    const result = resp.XAVResponse;

    if (result.ValidAddressIndicator) {
      console.log('Address is valid!');
    } else if (result.AmbiguousAddressIndicator) {
      console.log('Multiple candidates:', result.Candidate);
    } else {
      console.log('No candidates / invalid address');
    }
  } catch (err) {
    handleUpsError(err);
  }
})();

// --- Error helper ---
import { AxiosError } from 'axios';
import { UpsErrorResponse } from './types/ups-address-validation';

function handleUpsError(err: unknown) {
  if (axios.isAxiosError(err)) {
    const e = err as AxiosError<UpsErrorResponse>;
    console.error('UPS API error:', e.response?.data?.response?.errors ?? e.message);
  } else {
    console.error('Unknown error', err);
  }
}
```

---

## 6. Error Handling Patterns
UPS uses HTTP status codes **and** a JSON body:

* `400` – invalid request (missing fields, bad enum, etc.)  
* `401` – bad/expired token  
* `403` – blocked merchant  
* `429` – rate-limit (back-off required)  
* `500` – UPS internal error  

Always wrap calls in `try/catch` and:

1. Inspect `error.response.status`  
2. Map `error.response.data.response.errors` to your domain errors  
3. Implement **retry with exponential back-off** for `429` & `5xx`  

---

## 7. Best Practices for Production

1. **Cache OAuth tokens** in memory or Redis to minimise auth calls.  
2. **Validate input client-side** to cut unnecessary UPS traffic (e.g., enforce 2-char state).  
3. Centralise UPS integration in a tiny service / module (as shown) for easier upgrades.  
4. **Graceful degradation**: if UPS is unavailable, allow checkout but flag address for manual review.  
5. Respect **rate limits** – UPS may throttle; implement circuit-breaker or queue.  
6. Keep **CIE vs Production** credentials & base URLs in environment variables.  
7. Log `TransactionReference.CustomerContext` in requests so you can correlate UPS logs with your own.  
8. When showing candidates to end-users, highlight differences (street, ZIP) and let user pick.  
9. Upgrade to new versions promptly; `v1` is deprecated (array vs object variance).  
10. Monitor `AddressClassification` — residential surcharges can be reduced if you downgrade to commercial.  

---

Happy shipping!
