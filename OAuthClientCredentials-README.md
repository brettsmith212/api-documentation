# UPS OAuth Client Credentials API
Comprehensive README for Node.js + TypeScript Integrators  

---

## 1. What Is the OAuth Client Credentials API?
The **OAuth Client Credentials API** retrieves OAuth Bearer tokens when the integration owner is also the UPS shipper. This is the most common authentication method for server-to-server integrations where your application acts on behalf of your own UPS account.

Key characteristics:
* **Server-to-server** authentication (no user interaction required)
* **Short-lived tokens** (approximately 1 hour)  
* **No refresh tokens** (must re-authenticate when expired)
* **Requires UPS account number** for some APIs

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/security/v1/oauth/token` | Generate access token |

### 2.1 Request Body (form-encoded)
```
grant_type=client_credentials
```

### 2.2 Headers
* `Authorization: Basic base64(client_id:client_secret)`
* `Content-Type: application/x-www-form-urlencoded`
* `x-merchant-id: YOUR_UPS_ACCOUNT_NUMBER` (optional, required for some APIs)

---

## 3. TypeScript Interfaces

```ts
// Token Response
export interface OAuthTokenResponse {
  token_type: string;
  issued_at: string;
  client_id: string;
  access_token: string;
  scope: string;
  expires_in: string; // seconds as string
  refresh_count: string;
  status: string;
}

// Error Response
export interface OAuthErrorMessage {
  code: string;
  message: string;
}

export interface OAuthErrorResponse {
  response: {
    errors: OAuthErrorMessage[];
  };
}

// Client Configuration
export interface UpsOAuthClientConfig {
  clientId: string;
  clientSecret: string;
  merchantId?: string; // UPS account number
  baseURL?: string;
}
```

---

## 4. Production-Ready Node.js Implementation

### 4.1 Install Dependencies

```bash
npm i axios qs
npm i -D @types/node
```

### 4.2 `upsOAuthClient.ts` - OAuth Client

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import qs from 'qs';
import {
  OAuthTokenResponse,
  OAuthErrorResponse,
  UpsOAuthClientConfig
} from './types/ups-oauth';

export class UpsOAuthClient {
  private axios: AxiosInstance;
  private token?: OAuthTokenResponse;
  private tokenFetchedAt = 0;

  constructor(private readonly config: UpsOAuthClientConfig) {
    this.axios = axios.create({
      baseURL: config.baseURL ?? 'https://wwwcie.ups.com',
      timeout: 15_000
    });
  }

  /** Get (and cache) access token */
  async getAccessToken(): Promise<string> {
    const now = Date.now();
    
    // Check if we have a valid cached token (with 60 second buffer)
    if (this.token && now - this.tokenFetchedAt < (parseInt(this.token.expires_in) - 60) * 1000) {
      return this.token.access_token;
    }

    // Generate new token
    const auth = Buffer.from(
      `${this.config.clientId}:${this.config.clientSecret}`
    ).toString('base64');

    const headers: Record<string, string> = {
      'Content-Type': 'application/x-www-form-urlencoded',
      Authorization: `Basic ${auth}`
    };

    // Add merchant ID if provided
    if (this.config.merchantId) {
      headers['x-merchant-id'] = this.config.merchantId;
    }

    try {
      const { data } = await this.axios.post<OAuthTokenResponse>(
        '/security/v1/oauth/token',
        qs.stringify({ grant_type: 'client_credentials' }),
        { headers }
      );

      this.token = data;
      this.tokenFetchedAt = now;
      return data.access_token;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }

  /** Clear cached token (force refresh on next call) */
  clearToken(): void {
    this.token = undefined;
    this.tokenFetchedAt = 0;
  }

  /** Get token info (without refreshing) */
  getTokenInfo(): OAuthTokenResponse | undefined {
    return this.token;
  }

  /** Check if token is expired */
  isTokenExpired(): boolean {
    if (!this.token) return true;
    
    const now = Date.now();
    const expiryTime = this.tokenFetchedAt + (parseInt(this.token.expires_in) * 1000);
    return now >= expiryTime;
  }

  private handleAuthError(error: unknown): void {
    if (axios.isAxiosError(error)) {
      const e = error as AxiosError<OAuthErrorResponse>;
      const errorData = e.response?.data?.response?.errors;
      
      if (errorData) {
        console.error('UPS OAuth Error:', errorData);
        throw new Error(`OAuth failed: ${errorData.map(err => err.message).join(', ')}`);
      }
    }
    
    console.error('Unknown OAuth error:', error);
    throw new Error('OAuth authentication failed');
  }
}
```

### 4.3 Usage Example

```ts
import { UpsOAuthClient } from './upsOAuthClient';

// Initialize client
const oauthClient = new UpsOAuthClient({
  clientId: process.env.UPS_CLIENT_ID!,
  clientSecret: process.env.UPS_CLIENT_SECRET!,
  merchantId: process.env.UPS_MERCHANT_ID, // 6-digit account number
  baseURL: process.env.NODE_ENV === 'production' 
    ? 'https://onlinetools.ups.com'
    : 'https://wwwcie.ups.com'
});

// Get token and use with other APIs
async function makeUpsApiCall() {
  try {
    const token = await oauthClient.getAccessToken();
    
    // Use token for other UPS API calls
    const response = await axios.get('https://wwwcie.ups.com/api/some-endpoint', {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    });
    
    return response.data;
  } catch (error) {
    console.error('API call failed:', error);
    throw error;
  }
}

// Example: Create a higher-level UPS API client
class UpsApiClient {
  constructor(private oauthClient: UpsOAuthClient) {}

  async makeAuthenticatedRequest(endpoint: string, options: any = {}) {
    const token = await this.oauthClient.getAccessToken();
    
    return axios({
      ...options,
      url: endpoint,
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
        ...options.headers
      }
    });
  }
}
```

---

## 5. Error Handling

Common HTTP status codes:
* `200` - Success
* `400` - Invalid request (check client credentials)
* `401` - Unauthorized (invalid credentials)
* `403` - Blocked merchant
* `429` - Rate limit exceeded

```ts
function handleOAuthError(error: unknown) {
  if (axios.isAxiosError(error)) {
    const status = error.response?.status;
    const errorData = error.response?.data?.response?.errors;
    
    switch (status) {
      case 400:
        console.error('Invalid request:', errorData);
        break;
      case 401:
        console.error('Invalid credentials - check client ID/secret');
        break;
      case 403:
        console.error('Merchant blocked');
        break;
      case 429:
        console.error('Rate limit exceeded - implement backoff');
        break;
      default:
        console.error('OAuth error:', errorData || error.message);
    }
  }
}
```

---

## 6. Best Practices

1. **Cache tokens** - Don't fetch a new token for every API call
2. **Handle expiration** - Implement automatic token refresh
3. **Secure credentials** - Never hardcode client secrets
4. **Rate limiting** - Implement exponential backoff for 429 errors
5. **Environment management** - Use different credentials for test/production
6. **Error handling** - Always handle authentication failures gracefully
7. **Logging** - Log authentication events for debugging
8. **Timeout handling** - Set reasonable timeouts for token requests

### 6.1 Environment Variables

```bash
# .env
UPS_CLIENT_ID=your_client_id
UPS_CLIENT_SECRET=your_client_secret
UPS_MERCHANT_ID=123456  # Your 6-digit UPS account number
NODE_ENV=development    # or production
```

### 6.2 Production Monitoring

```ts
// Monitor token usage and renewal
class TokenMonitor {
  constructor(private oauthClient: UpsOAuthClient) {}

  async monitorTokenHealth() {
    const tokenInfo = this.oauthClient.getTokenInfo();
    
    if (tokenInfo) {
      const expiresIn = parseInt(tokenInfo.expires_in);
      const timeUntilExpiry = expiresIn - (Date.now() - this.tokenFetchedAt) / 1000;
      
      if (timeUntilExpiry < 300) { // 5 minutes warning
        console.warn('Token expiring soon, consider refreshing');
      }
    }
  }
}
```

---

## 7. Testing

```ts
// Example test
import { UpsOAuthClient } from './upsOAuthClient';

describe('UPS OAuth Client', () => {
  let client: UpsOAuthClient;

  beforeEach(() => {
    client = new UpsOAuthClient({
      clientId: 'test_client',
      clientSecret: 'test_secret',
      baseURL: 'https://wwwcie.ups.com'
    });
  });

  it('should get access token', async () => {
    const token = await client.getAccessToken();
    expect(token).toBeDefined();
    expect(typeof token).toBe('string');
  });

  it('should cache tokens', async () => {
    const token1 = await client.getAccessToken();
    const token2 = await client.getAccessToken();
    expect(token1).toBe(token2);
  });
});
```

---

Happy shipping!
