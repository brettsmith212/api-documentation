# UPS OAuth Authorization Code API
Comprehensive README for Node.js + TypeScript Integrators  

---

## 1. What Is the OAuth Authorization Code API?
The **OAuth Authorization Code API** enables third-party applications to integrate UPS services on behalf of UPS customers. Unlike Client Credentials, this flow requires user interaction and provides **refresh tokens** for long-term access.

Key characteristics:
* **User-interactive** flow (redirects to UPS login)
* **Long-lived access** via refresh tokens
* **PKCE enhancement** for improved security
* **Suitable for** web applications serving multiple UPS customers

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/v1/oauth/authorize` | Initiate authorization flow |
| `POST` | `/v1/oauth/token` | Exchange code for tokens |
| `POST` | `/v1/oauth/refresh` | Refresh expired access token |

---

## 3. Authorization Flow Types

### 3.1 Standard Authorization Code Flow
1. Redirect user to `/authorize` endpoint
2. User logs in and grants permission
3. UPS redirects back with authorization code
4. Exchange code for access + refresh tokens

### 3.2 PKCE-Enhanced Flow (Recommended)
1. Generate `code_verifier` and `code_challenge`
2. Redirect user to `/authorize` with challenge
3. User logs in and grants permission
4. Exchange code + verifier for tokens (no Basic auth required)

---

## 4. TypeScript Interfaces

```ts
// PKCE Helper
export interface PKCEPair {
  codeVerifier: string;
  codeChallenge: string;
}

// Authorization Request
export interface AuthorizationParams {
  client_id: string;
  redirect_uri: string;
  response_type: 'code';
  state?: string;
  scope?: string;
  code_challenge?: string; // For PKCE
}

// Token Request (Standard)
export interface TokenRequest {
  grant_type: 'authorization_code';
  code: string;
  redirect_uri: string;
}

// Token Request (PKCE)
export interface PKCETokenRequest extends TokenRequest {
  code_verifier: string;
  client_id: string;
}

// Token Response
export interface TokenResponse {
  access_token: string;
  refresh_token: string;
  token_type: 'Bearer';
  expires_in: string; // seconds
  refresh_token_expires_in: string;
  refresh_token_status: string;
  issued_at: string;
  client_id: string;
  scope: string;
  refresh_token_issued_at: string;
  refresh_count: string;
  status: string;
}

// Refresh Token Request
export interface RefreshTokenRequest {
  grant_type: 'refresh_token';
  refresh_token: string;
}

// Error Response
export interface OAuthError {
  code: string;
  message: string;
}

export interface OAuthErrorResponse {
  response: {
    errors: OAuthError[];
  };
}
```

---

## 5. Production-Ready Node.js Implementation

### 5.1 Install Dependencies

```bash
npm i axios qs crypto
npm i -D @types/node
```

### 5.2 `upsOAuthAuthCode.ts` - Authorization Code Client

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import qs from 'qs';
import crypto from 'crypto';
import {
  PKCEPair,
  AuthorizationParams,
  TokenResponse,
  PKCETokenRequest,
  RefreshTokenRequest,
  OAuthErrorResponse
} from './types/ups-oauth-authcode';

export interface UpsOAuthConfig {
  clientId: string;
  clientSecret?: string; // Not needed for PKCE
  redirectUri: string;
  baseURL?: string;
  scope?: string;
}

export class UpsOAuthAuthCodeClient {
  private axios: AxiosInstance;
  private tokens: Map<string, TokenResponse> = new Map(); // userId -> tokens

  constructor(private readonly config: UpsOAuthConfig) {
    this.axios = axios.create({
      baseURL: config.baseURL ?? 'https://wwwcie.ups.com',
      timeout: 15_000
    });
  }

  /** Generate PKCE code verifier and challenge */
  generatePKCE(): PKCEPair {
    const codeVerifier = crypto.randomBytes(32).toString('base64url');
    const codeChallenge = crypto
      .createHash('sha256')
      .update(codeVerifier)
      .digest('base64url');

    return { codeVerifier, codeChallenge };
  }

  /** Generate authorization URL (PKCE recommended) */
  getAuthorizationUrl(state?: string, usePKCE: boolean = true): { url: string; codeVerifier?: string } {
    const params: AuthorizationParams = {
      client_id: this.config.clientId,
      redirect_uri: this.config.redirectUri,
      response_type: 'code',
      state,
      scope: this.config.scope
    };

    let codeVerifier: string | undefined;
    
    if (usePKCE) {
      const pkce = this.generatePKCE();
      params.code_challenge = pkce.codeChallenge;
      codeVerifier = pkce.codeVerifier;
    }

    const queryString = new URLSearchParams(params as any).toString();
    const url = `${this.config.baseURL}/security/v1/oauth/authorize?${queryString}`;

    return { url, codeVerifier };
  }

  /** Exchange authorization code for tokens (PKCE flow) */
  async exchangeCodeForTokens(
    code: string,
    codeVerifier: string,
    userId?: string
  ): Promise<TokenResponse> {
    const tokenRequest: PKCETokenRequest = {
      grant_type: 'authorization_code',
      code,
      redirect_uri: this.config.redirectUri,
      code_verifier: codeVerifier,
      client_id: this.config.clientId
    };

    try {
      const { data } = await this.axios.post<TokenResponse>(
        '/security/v1/oauth/token',
        qs.stringify(tokenRequest),
        {
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
          }
        }
      );

      // Cache tokens if userId provided
      if (userId) {
        this.tokens.set(userId, data);
      }

      return data;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }

  /** Exchange authorization code for tokens (Standard flow) */
  async exchangeCodeForTokensStandard(
    code: string,
    userId?: string
  ): Promise<TokenResponse> {
    if (!this.config.clientSecret) {
      throw new Error('Client secret required for standard flow');
    }

    const tokenRequest = {
      grant_type: 'authorization_code',
      code,
      redirect_uri: this.config.redirectUri
    };

    const auth = Buffer.from(
      `${this.config.clientId}:${this.config.clientSecret}`
    ).toString('base64');

    try {
      const { data } = await this.axios.post<TokenResponse>(
        '/security/v1/oauth/token',
        qs.stringify(tokenRequest),
        {
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Authorization': `Basic ${auth}`
          }
        }
      );

      // Cache tokens if userId provided
      if (userId) {
        this.tokens.set(userId, data);
      }

      return data;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }

  /** Refresh access token */
  async refreshAccessToken(refreshToken: string, userId?: string): Promise<TokenResponse> {
    if (!this.config.clientSecret) {
      throw new Error('Client secret required for refresh');
    }

    const refreshRequest: RefreshTokenRequest = {
      grant_type: 'refresh_token',
      refresh_token: refreshToken
    };

    const auth = Buffer.from(
      `${this.config.clientId}:${this.config.clientSecret}`
    ).toString('base64');

    try {
      const { data } = await this.axios.post<TokenResponse>(
        '/security/v1/oauth/refresh',
        qs.stringify(refreshRequest),
        {
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Authorization': `Basic ${auth}`
          }
        }
      );

      // Update cached tokens
      if (userId) {
        this.tokens.set(userId, data);
      }

      return data;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }

  /** Get valid access token for user (auto-refresh if needed) */
  async getValidAccessToken(userId: string): Promise<string> {
    const tokens = this.tokens.get(userId);
    if (!tokens) {
      throw new Error('No tokens found for user. User must re-authenticate.');
    }

    // Check if access token is expired
    const now = Date.now();
    const issuedAt = parseInt(tokens.issued_at);
    const expiresIn = parseInt(tokens.expires_in) * 1000;
    const accessTokenExpired = now > (issuedAt + expiresIn - 60000); // 1 min buffer

    if (accessTokenExpired) {
      // Try to refresh
      const refreshTokenExpiresIn = parseInt(tokens.refresh_token_expires_in) * 1000;
      const refreshTokenIssued = parseInt(tokens.refresh_token_issued_at);
      const refreshTokenExpired = now > (refreshTokenIssued + refreshTokenExpiresIn);

      if (refreshTokenExpired) {
        throw new Error('Refresh token expired. User must re-authenticate.');
      }

      // Refresh the access token
      const newTokens = await this.refreshAccessToken(tokens.refresh_token, userId);
      return newTokens.access_token;
    }

    return tokens.access_token;
  }

  /** Remove tokens for user */
  revokeTokens(userId: string): void {
    this.tokens.delete(userId);
  }

  /** Check if user has valid tokens */
  hasValidTokens(userId: string): boolean {
    try {
      const tokens = this.tokens.get(userId);
      if (!tokens) return false;

      const now = Date.now();
      const refreshTokenIssued = parseInt(tokens.refresh_token_issued_at);
      const refreshTokenExpiresIn = parseInt(tokens.refresh_token_expires_in) * 1000;
      
      return now < (refreshTokenIssued + refreshTokenExpiresIn);
    } catch {
      return false;
    }
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

### 5.3 Express.js Integration Example

```ts
import express from 'express';
import { UpsOAuthAuthCodeClient } from './upsOAuthAuthCode';

const app = express();
const oauthClient = new UpsOAuthAuthCodeClient({
  clientId: process.env.UPS_CLIENT_ID!,
  clientSecret: process.env.UPS_CLIENT_SECRET!,
  redirectUri: 'https://yourapp.com/auth/ups/callback',
  baseURL: process.env.NODE_ENV === 'production' 
    ? 'https://onlinetools.ups.com'
    : 'https://wwwcie.ups.com'
});

// Session storage for PKCE verifiers (use Redis in production)
const sessionStore = new Map<string, { codeVerifier: string; userId: string }>();

// Initiate OAuth flow
app.get('/auth/ups/login', (req, res) => {
  const userId = req.query.userId as string;
  const state = crypto.randomBytes(16).toString('hex');
  
  const { url, codeVerifier } = oauthClient.getAuthorizationUrl(state, true);
  
  // Store code verifier for later use
  sessionStore.set(state, { codeVerifier!, userId });
  
  res.redirect(url);
});

// Handle OAuth callback
app.get('/auth/ups/callback', async (req, res) => {
  const { code, state, error } = req.query;
  
  if (error) {
    return res.status(400).json({ error: 'Authorization failed', details: error });
  }
  
  const sessionData = sessionStore.get(state as string);
  if (!sessionData) {
    return res.status(400).json({ error: 'Invalid state parameter' });
  }
  
  try {
    const tokens = await oauthClient.exchangeCodeForTokens(
      code as string,
      sessionData.codeVerifier,
      sessionData.userId
    );
    
    sessionStore.delete(state as string);
    
    res.json({ 
      message: 'Authorization successful',
      expires_in: tokens.expires_in
    });
  } catch (error) {
    console.error('Token exchange failed:', error);
    res.status(500).json({ error: 'Token exchange failed' });
  }
});

// Protected route example
app.get('/api/user/:userId/shipping', async (req, res) => {
  const userId = req.params.userId;
  
  try {
    const accessToken = await oauthClient.getValidAccessToken(userId);
    
    // Use access token to call UPS APIs
    const response = await axios.get('https://wwwcie.ups.com/api/shipments/v1/ship', {
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json'
      }
    });
    
    res.json(response.data);
  } catch (error) {
    if (error.message.includes('re-authenticate')) {
      res.status(401).json({ error: 'User must re-authenticate', requiresAuth: true });
    } else {
      res.status(500).json({ error: 'API call failed' });
    }
  }
});
```

---

## 6. Best Practices

1. **Use PKCE** - Always use PKCE enhancement for better security
2. **Secure state parameter** - Use cryptographically secure random state
3. **Token storage** - Store tokens securely (encrypted database, not cookies)
4. **Refresh handling** - Implement automatic token refresh
5. **Error handling** - Handle expired tokens gracefully
6. **User experience** - Provide clear authentication status
7. **Logging** - Log OAuth events for debugging
8. **Testing** - Test full OAuth flow in staging environment

### 6.1 Security Considerations

```ts
// Generate cryptographically secure state
function generateSecureState(): string {
  return crypto.randomBytes(32).toString('hex');
}

// Validate state parameter
function validateState(receivedState: string, expectedState: string): boolean {
  return crypto.timingSafeEqual(
    Buffer.from(receivedState, 'hex'),
    Buffer.from(expectedState, 'hex')
  );
}
```

### 6.2 Token Management

```ts
// Example token manager with database storage
class TokenManager {
  async storeTokens(userId: string, tokens: TokenResponse): Promise<void> {
    // Store in encrypted database
    await db.query(
      'INSERT INTO user_tokens (user_id, tokens, created_at) VALUES (?, ?, ?)',
      [userId, encrypt(JSON.stringify(tokens)), new Date()]
    );
  }

  async getTokens(userId: string): Promise<TokenResponse | null> {
    const result = await db.query(
      'SELECT tokens FROM user_tokens WHERE user_id = ?',
      [userId]
    );
    
    return result.length > 0 ? JSON.parse(decrypt(result[0].tokens)) : null;
  }
}
```

---

## 7. Testing

```ts
// Example test for OAuth flow
describe('UPS OAuth Authorization Code', () => {
  let client: UpsOAuthAuthCodeClient;

  beforeEach(() => {
    client = new UpsOAuthAuthCodeClient({
      clientId: 'test_client',
      clientSecret: 'test_secret',
      redirectUri: 'http://localhost:3000/callback',
      baseURL: 'https://wwwcie.ups.com'
    });
  });

  it('should generate PKCE pair', () => {
    const pkce = client.generatePKCE();
    expect(pkce.codeVerifier).toBeDefined();
    expect(pkce.codeChallenge).toBeDefined();
  });

  it('should generate authorization URL', () => {
    const { url, codeVerifier } = client.getAuthorizationUrl('test_state');
    expect(url).toContain('client_id=test_client');
    expect(url).toContain('response_type=code');
    expect(codeVerifier).toBeDefined();
  });
});
```

---

Happy shipping!
