# UPS Time In Transit API
Comprehensive README for Node.js + TypeScript Integrators

---

## 1. What Is the Time In Transit API?

The **Time In Transit API** provides estimated delivery times for various UPS shipping services between specified locations. It helps businesses provide accurate delivery estimates to customers and optimize logistics planning.

Key business values:
* **Enhanced Customer Experience** - Provide accurate delivery estimates to customers
* **Operational Efficiency** - Optimize logistics planning with precise transit times
* **Service Selection** - Compare delivery times across different UPS services
* **International Support** - Handle both domestic and international shipments

The API supports both small package and freight shipments with detailed transit time calculations including business days, holidays, and service guarantees.

Current stable version: `v1`

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/shipments/{version}/transittimes` | Get estimated delivery times |

### 2.1 Path Parameters
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `version` | string | `v1` | API version |

### 2.2 Required Headers
| Header | Type | Description |
|--------|------|-------------|
| `Authorization` | string | Bearer token from OAuth2 |
| `transId` | string | Unique transaction ID (32 characters) |
| `transactionSrc` | string | Source application identifier (≤512 chars) |

### 2.3 Request Body
The request body contains origin/destination information, shipment details, and service preferences.

---

## 3. TypeScript Interfaces

```ts
// OAuth Token
export interface OAuthToken {
  access_token: string;
  token_type: 'Bearer';
  expires_in: number;
}

// Request Types
export interface TimeInTransitRequest {
  originCountryCode: string;
  originStateProvince?: string;
  originCityName?: string;
  originTownName?: string;
  originPostalCode?: string;
  destinationCountryCode?: string;
  destinationStateProvince?: string;
  destinationCityName?: string;
  destinationTownName?: string;
  destinationPostalCode?: string;
  residentialIndicator?: '01' | '02'; // 01=Residential, 02=Commercial
  shipDate?: string; // YYYY-MM-DD format
  shipTime?: string; // HH:MM:SS format
  weight?: number;
  weightUnitOfMeasure?: 'LBS' | 'KGS';
  shipmentContentsValue?: number;
  shipmentContentsCurrencyCode?: string;
  billType?: '02' | '03' | '04'; // 02=Document, 03=Non-Document, 04=WWEF
  avvFlag?: boolean;
  numberOfPackages?: number;
  dropOffAtFacilityIndicator?: 0 | 1;
  holdForPickupIndicator?: 0 | 1;
  returnUnfilterdServices?: boolean;
  maxList?: number;
}

// Response Types
export interface TimeInTransitResponse {
  validationList?: ValidationList;
  destinationPickList?: CandidateAddress[];
  originPickList?: CandidateAddress[];
  emsResponse?: EmsResponse;
}

export interface ValidationList {
  invalidFieldList?: string[];
  invalidFieldListCodes?: string[];
  destinationAmbiguous?: boolean;
  originAmbiguous?: boolean;
}

export interface CandidateAddress {
  countryName: string;
  countryCode: string;
  stateProvince?: string;
  city?: string;
  town?: string;
  postalCode?: string;
  postalCodeLow?: string;
  postalCodeHigh?: string;
}

export interface EmsResponse {
  shipDate: string;
  shipTime: string;
  serviceLevel: string;
  billType: string;
  dutyType?: string;
  residentialIndicator: string;
  destinationCountryName: string;
  destinationCountryCode: string;
  destinationPostalCode?: string;
  destinationPostalCodeLow?: string;
  destinationPostalCodeHigh?: string;
  destinationStateProvince?: string;
  destinationCityName?: string;
  originCountryName: string;
  originCountryCode: string;
  originPostalCode?: string;
  originPostalCodeLow?: string;
  originPostalCodeHigh?: string;
  originStateProvince?: string;
  originCityName?: string;
  weight?: string;
  weightUnitOfMeasure?: string;
  shipmentContentsValue?: string;
  shipmentContentsCurrencyCode?: string;
  guaranteeSuspended: boolean;
  numberOfServices: number;
  services: Service[];
}

export interface Service {
  serviceLevel: string;
  serviceLevelDescription: string;
  shipDate: string;
  deliveryDate: string;
  commitTime: string;
  deliveryTime: string;
  deliveryDayOfWeek: 'MON' | 'TUE' | 'WED' | 'THU' | 'FRI' | 'SAT';
  nextDayPickupIndicator?: string;
  saturdayPickupIndicator?: string;
  saturdayDeliveryDate?: string;
  saturdayDeliveryTime?: string;
  serviceRemarksText?: string;
  guaranteeIndicator: '0' | '1'; // 0=Not Guaranteed, 1=Guaranteed
  totalTransitDays?: number;
  businessTransitDays?: number;
  restDaysCount?: number;
  holidayCount?: number;
  delayCount?: number;
  pickupDate?: string;
  pickupTime?: string;
  cstccutoffTime?: string;
  poddate?: string;
  poddays?: number;
}

// Error Types
export interface TimeInTransitError {
  code: string;
  message: string;
}

export interface TimeInTransitErrorResponse {
  errors: TimeInTransitError[];
}
```

---

## 4. Authentication (OAuth2 Client Credentials)

```ts
import axios from 'axios';
import qs from 'qs';

export class UpsOAuthClient {
  private token?: OAuthToken;
  private tokenFetchedAt = 0;

  constructor(
    private readonly clientId: string,
    private readonly clientSecret: string,
    private readonly baseURL = 'https://wwwcie.ups.com'
  ) {}

  async getAccessToken(): Promise<string> {
    const now = Date.now();
    
    // Return cached token if still valid (with 60s buffer)
    if (this.token && now - this.tokenFetchedAt < (this.token.expires_in - 60) * 1000) {
      return this.token.access_token;
    }

    const auth = Buffer.from(`${this.clientId}:${this.clientSecret}`).toString('base64');
    
    try {
      const { data } = await axios.post<OAuthToken>(
        `${this.baseURL}/security/v1/oauth/token`,
        qs.stringify({ grant_type: 'client_credentials' }),
        {
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Authorization': `Basic ${auth}`
          }
        }
      );

      this.token = data;
      this.tokenFetchedAt = now;
      return data.access_token;
    } catch (error) {
      throw new Error(`OAuth token request failed: ${error}`);
    }
  }
}
```

---

## 5. Complete Node.js Implementation

### 5.1 Install Dependencies

```bash
npm install axios uuid
npm install -D @types/node @types/uuid
```

### 5.2 `upsTimeInTransitClient.ts`

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import { v4 as uuidv4 } from 'uuid';
import {
  TimeInTransitRequest,
  TimeInTransitResponse,
  TimeInTransitErrorResponse,
  Service
} from './types/ups-time-in-transit';

export interface UpsTimeInTransitClientConfig {
  baseURL?: string;
  timeout?: number;
  transactionSrc?: string;
}

export class UpsTimeInTransitClient {
  private axios: AxiosInstance;
  private transactionSrc: string;

  constructor(
    private getAccessToken: () => Promise<string>,
    config: UpsTimeInTransitClientConfig = {}
  ) {
    this.axios = axios.create({
      baseURL: config.baseURL ?? 'https://wwwcie.ups.com/api',
      timeout: config.timeout ?? 30_000
    });

    this.transactionSrc = config.transactionSrc ?? 'TIME_IN_TRANSIT_API';
  }

  /**
   * Get estimated delivery times for shipment
   */
  async getTransitTimes(
    request: TimeInTransitRequest,
    transId?: string
  ): Promise<TimeInTransitResponse> {
    const accessToken = await this.getAccessToken();
    const transactionId = transId ?? uuidv4();

    try {
      const { data } = await this.axios.post<TimeInTransitResponse>(
        '/shipments/v1/transittimes',
        request,
        {
          headers: {
            'Authorization': `Bearer ${accessToken}`,
            'transId': transactionId,
            'transactionSrc': this.transactionSrc
          }
        }
      );

      return data;
    } catch (error) {
      this.handleError(error, request);
      throw error;
    }
  }

  /**
   * Get fastest delivery service
   */
  async getFastestService(request: TimeInTransitRequest): Promise<Service | null> {
    const response = await this.getTransitTimes(request);
    
    if (!response.emsResponse?.services?.length) {
      return null;
    }

    return response.emsResponse.services.reduce((fastest, current) => {
      const fastestDays = fastest.businessTransitDays ?? Infinity;
      const currentDays = current.businessTransitDays ?? Infinity;
      
      return currentDays < fastestDays ? current : fastest;
    });
  }

  /**
   * Get all available services sorted by transit time
   */
  async getServicesSortedBySpeed(request: TimeInTransitRequest): Promise<Service[]> {
    const response = await this.getTransitTimes(request);
    
    if (!response.emsResponse?.services?.length) {
      return [];
    }

    return response.emsResponse.services.sort((a, b) => {
      const aDays = a.businessTransitDays ?? Infinity;
      const bDays = b.businessTransitDays ?? Infinity;
      return aDays - bDays;
    });
  }

  /**
   * Get guaranteed services only
   */
  async getGuaranteedServices(request: TimeInTransitRequest): Promise<Service[]> {
    const response = await this.getTransitTimes(request);
    
    if (!response.emsResponse?.services?.length) {
      return [];
    }

    return response.emsResponse.services.filter(service => 
      service.guaranteeIndicator === '1'
    );
  }

  /**
   * Check if delivery is guaranteed on specific date
   */
  async isDeliveryGuaranteed(
    request: TimeInTransitRequest,
    targetDate: string
  ): Promise<boolean> {
    const response = await this.getTransitTimes(request);
    
    if (!response.emsResponse?.services?.length) {
      return false;
    }

    return response.emsResponse.services.some(service => 
      service.deliveryDate === targetDate && service.guaranteeIndicator === '1'
    );
  }

  /**
   * Get transit time summary
   */
  async getTransitSummary(request: TimeInTransitRequest) {
    const response = await this.getTransitTimes(request);
    
    if (!response.emsResponse?.services?.length) {
      return {
        servicesFound: 0,
        fastestTransitDays: null,
        guaranteedServices: 0,
        guaranteeSuspended: false
      };
    }

    const services = response.emsResponse.services;
    const guaranteedServices = services.filter(s => s.guaranteeIndicator === '1');
    const fastestService = services.reduce((fastest, current) => {
      const fastestDays = fastest.businessTransitDays ?? Infinity;
      const currentDays = current.businessTransitDays ?? Infinity;
      return currentDays < fastestDays ? current : fastest;
    });

    return {
      servicesFound: services.length,
      fastestTransitDays: fastestService.businessTransitDays,
      guaranteedServices: guaranteedServices.length,
      guaranteeSuspended: response.emsResponse.guaranteeSuspended,
      services: services.map(s => ({
        code: s.serviceLevel,
        description: s.serviceLevelDescription,
        deliveryDate: s.deliveryDate,
        businessDays: s.businessTransitDays,
        guaranteed: s.guaranteeIndicator === '1'
      }))
    };
  }

  private handleError(error: unknown, request: TimeInTransitRequest): void {
    if (axios.isAxiosError(error)) {
      const e = error as AxiosError<TimeInTransitErrorResponse>;
      const status = e.response?.status;
      const errorData = e.response?.data?.errors;

      switch (status) {
        case 400:
          const fieldErrors = errorData?.map(err => err.message).join(', ');
          throw new Error(`Invalid request: ${fieldErrors || 'Check required fields'}`);
        case 401:
          throw new Error('Unauthorized - check OAuth token');
        case 403:
          throw new Error('Forbidden - check account permissions');
        case 429:
          throw new Error('Rate limit exceeded - please retry later');
        case 500:
          throw new Error('UPS service temporarily unavailable');
        default:
          const message = errorData?.map(err => err.message).join(', ') || 'Unknown error';
          throw new Error(`Transit time request failed: ${message}`);
      }
    }

    throw new Error(`Transit time request failed: ${error}`);
  }
}
```

### 5.3 Usage Examples

```ts
import { UpsTimeInTransitClient } from './upsTimeInTransitClient';
import { UpsOAuthClient } from './upsOAuthClient';

// Setup OAuth client
const oauthClient = new UpsOAuthClient(
  process.env.UPS_CLIENT_ID!,
  process.env.UPS_CLIENT_SECRET!,
  process.env.NODE_ENV === 'production' 
    ? 'https://onlinetools.ups.com'
    : 'https://wwwcie.ups.com'
);

// Create Time In Transit client
const transitClient = new UpsTimeInTransitClient(
  () => oauthClient.getAccessToken(),
  {
    baseURL: process.env.NODE_ENV === 'production' 
      ? 'https://onlinetools.ups.com/api'
      : 'https://wwwcie.ups.com/api',
    transactionSrc: 'MyShippingApp'
  }
);

// Basic transit time request
async function getTransitTimes() {
  const request: TimeInTransitRequest = {
    originCountryCode: 'US',
    originStateProvince: 'CA',
    originCityName: 'San Francisco',
    originPostalCode: '94105',
    destinationCountryCode: 'US',
    destinationStateProvince: 'NY',
    destinationCityName: 'New York',
    destinationPostalCode: '10001',
    residentialIndicator: '02', // Commercial
    shipDate: '2024-01-15',
    weight: 5.0,
    weightUnitOfMeasure: 'LBS',
    numberOfPackages: 1,
    billType: '03' // Non-Document
  };

  try {
    const response = await transitClient.getTransitTimes(request);
    
    if (response.emsResponse?.services) {
      console.log('Available services:');
      response.emsResponse.services.forEach(service => {
        console.log(`${service.serviceLevelDescription}:`);
        console.log(`  Delivery Date: ${service.deliveryDate}`);
        console.log(`  Business Days: ${service.businessTransitDays}`);
        console.log(`  Guaranteed: ${service.guaranteeIndicator === '1' ? 'Yes' : 'No'}`);
      });
    }
  } catch (error) {
    console.error('Transit time request failed:', error.message);
  }
}

// Get fastest service
async function getFastestDelivery() {
  const request: TimeInTransitRequest = {
    originCountryCode: 'US',
    originPostalCode: '90210',
    destinationCountryCode: 'US',
    destinationPostalCode: '10001',
    weight: 2.5,
    weightUnitOfMeasure: 'LBS'
  };

  try {
    const fastestService = await transitClient.getFastestService(request);
    
    if (fastestService) {
      console.log(`Fastest option: ${fastestService.serviceLevelDescription}`);
      console.log(`Delivery date: ${fastestService.deliveryDate}`);
      console.log(`Business days: ${fastestService.businessTransitDays}`);
    }
  } catch (error) {
    console.error('Failed to get fastest service:', error.message);
  }
}

// International shipment example
async function getInternationalTransitTimes() {
  const request: TimeInTransitRequest = {
    originCountryCode: 'US',
    originPostalCode: '10001',
    destinationCountryCode: 'GB',
    destinationPostalCode: 'SW1A 1AA',
    weight: 3.0,
    weightUnitOfMeasure: 'LBS',
    shipmentContentsValue: 100.00,
    shipmentContentsCurrencyCode: 'USD',
    billType: '03' // Non-Document
  };

  try {
    const summary = await transitClient.getTransitSummary(request);
    
    console.log(`Found ${summary.servicesFound} services`);
    console.log(`Fastest delivery: ${summary.fastestTransitDays} business days`);
    console.log(`Guaranteed services: ${summary.guaranteedServices}`);
    
    if (summary.guaranteeSuspended) {
      console.log('⚠️  Service guarantees are suspended (peak season)');
    }
  } catch (error) {
    console.error('International transit request failed:', error.message);
  }
}

// Check specific delivery date
async function checkDeliveryDate() {
  const request: TimeInTransitRequest = {
    originCountryCode: 'US',
    originPostalCode: '94105',
    destinationCountryCode: 'US',
    destinationPostalCode: '33101',
    shipDate: '2024-01-15',
    weight: 1.5,
    weightUnitOfMeasure: 'LBS'
  };

  try {
    const isGuaranteed = await transitClient.isDeliveryGuaranteed(request, '2024-01-17');
    
    console.log(`Guaranteed delivery by 2024-01-17: ${isGuaranteed ? 'Yes' : 'No'}`);
  } catch (error) {
    console.error('Delivery date check failed:', error.message);
  }
}
```

---

## 6. Error Handling

```ts
export class TransitTimeError extends Error {
  constructor(
    message: string,
    public code?: string,
    public statusCode?: number
  ) {
    super(message);
    this.name = 'TransitTimeError';
  }
}

export function handleTransitTimeError(error: unknown): never {
  if (error instanceof TransitTimeError) {
    throw error;
  }
  
  if (axios.isAxiosError(error)) {
    const status = error.response?.status;
    const errorData = error.response?.data?.errors;
    
    switch (status) {
      case 400:
        throw new TransitTimeError(
          'Invalid request parameters',
          'INVALID_REQUEST',
          400
        );
      case 401:
        throw new TransitTimeError(
          'Authentication failed',
          'UNAUTHORIZED',
          401
        );
      case 403:
        throw new TransitTimeError(
          'Access forbidden',
          'FORBIDDEN',
          403
        );
      case 429:
        throw new TransitTimeError(
          'Rate limit exceeded',
          'RATE_LIMIT',
          429
        );
      case 500:
        throw new TransitTimeError(
          'UPS service unavailable',
          'SERVICE_UNAVAILABLE',
          500
        );
      default:
        const message = errorData?.map(err => err.message).join(', ') || 'Unknown error';
        throw new TransitTimeError(message, 'UNKNOWN_ERROR', status);
    }
  }
  
  throw new TransitTimeError(`Unexpected error: ${error}`);
}
```

---

## 7. Best Practices for Transit Time Calculations

### 7.1 Request Optimization

1. **Provide complete addresses** - Include city, state, and postal code when possible
2. **Use accurate ship dates** - Within 60 days future, 35 days past
3. **Specify residential indicator** - Affects pickup and delivery options
4. **Include weight and dimensions** - Required for international shipments
5. **Set appropriate bill type** - Document vs Non-Document affects transit times

### 7.2 Response Handling

```ts
// Robust response validation
function validateTransitResponse(response: TimeInTransitResponse): boolean {
  // Check for address ambiguity
  if (response.validationList?.destinationAmbiguous || 
      response.validationList?.originAmbiguous) {
    console.log('Address clarification needed');
    return false;
  }

  // Check for invalid fields
  if (response.validationList?.invalidFieldList?.length) {
    console.log('Invalid fields:', response.validationList.invalidFieldList);
    return false;
  }

  // Verify services are available
  if (!response.emsResponse?.services?.length) {
    console.log('No services available for this route');
    return false;
  }

  return true;
}
```

### 7.3 Caching Strategy

```ts
// Cache transit times to reduce API calls
class TransitTimeCache {
  private cache = new Map<string, { data: TimeInTransitResponse; expires: number }>();
  private readonly ttl = 4 * 60 * 60 * 1000; // 4 hours

  private getCacheKey(request: TimeInTransitRequest): string {
    const key = {
      origin: `${request.originCountryCode}-${request.originPostalCode}`,
      destination: `${request.destinationCountryCode}-${request.destinationPostalCode}`,
      date: request.shipDate,
      weight: request.weight,
      type: request.billType
    };
    return JSON.stringify(key);
  }

  get(request: TimeInTransitRequest): TimeInTransitResponse | null {
    const key = this.getCacheKey(request);
    const cached = this.cache.get(key);
    
    if (cached && Date.now() < cached.expires) {
      return cached.data;
    }
    
    this.cache.delete(key);
    return null;
  }

  set(request: TimeInTransitRequest, data: TimeInTransitResponse): void {
    const key = this.getCacheKey(request);
    this.cache.set(key, {
      data,
      expires: Date.now() + this.ttl
    });
  }

  clear(): void {
    this.cache.clear();
  }
}
```

### 7.4 Rate Limiting

```ts
// Implement rate limiting to avoid API throttling
class ApiRateLimiter {
  private requests: number[] = [];
  private readonly maxRequests = 100;
  private readonly timeWindow = 60000; // 1 minute

  async waitForSlot(): Promise<void> {
    const now = Date.now();
    
    // Remove old requests
    this.requests = this.requests.filter(time => now - time < this.timeWindow);
    
    if (this.requests.length >= this.maxRequests) {
      const waitTime = this.timeWindow - (now - this.requests[0]);
      await new Promise(resolve => setTimeout(resolve, waitTime));
      return this.waitForSlot();
    }
    
    this.requests.push(now);
  }
}
```

### 7.5 Bulk Processing

```ts
// Process multiple transit time requests efficiently
async function processMultipleTransitRequests(
  requests: TimeInTransitRequest[],
  client: UpsTimeInTransitClient
): Promise<Map<string, TimeInTransitResponse | Error>> {
  const results = new Map<string, TimeInTransitResponse | Error>();
  const rateLimiter = new ApiRateLimiter();
  
  // Process in batches to avoid overwhelming the API
  const batchSize = 5;
  
  for (let i = 0; i < requests.length; i += batchSize) {
    const batch = requests.slice(i, i + batchSize);
    
    const promises = batch.map(async (request, index) => {
      const key = `${i + index}`;
      
      try {
        await rateLimiter.waitForSlot();
        const response = await client.getTransitTimes(request);
        results.set(key, response);
      } catch (error) {
        results.set(key, error as Error);
      }
    });
    
    await Promise.all(promises);
    
    // Small delay between batches
    if (i + batchSize < requests.length) {
      await new Promise(resolve => setTimeout(resolve, 200));
    }
  }
  
  return results;
}
```

### 7.6 Integration Tips

1. **Display delivery estimates prominently** - Show estimated delivery dates in checkout
2. **Handle guarantee suspensions** - Inform customers during peak seasons
3. **Update estimates regularly** - Transit times can change due to weather, holidays
4. **Provide service options** - Let customers choose between speed and cost
5. **Handle address ambiguity** - Provide address correction suggestions
6. **Monitor API performance** - Track response times and error rates
7. **Implement fallback logic** - Have backup plans when API is unavailable
8. **Use appropriate locales** - Support international customers with localized dates

---

## 8. Testing

```ts
// Unit tests for transit time client
describe('UpsTimeInTransitClient', () => {
  let client: UpsTimeInTransitClient;
  let mockGetToken: jest.Mock;

  beforeEach(() => {
    mockGetToken = jest.fn().mockResolvedValue('mock-token');
    client = new UpsTimeInTransitClient(mockGetToken);
  });

  it('should get transit times successfully', async () => {
    const request: TimeInTransitRequest = {
      originCountryCode: 'US',
      originPostalCode: '94105',
      destinationCountryCode: 'US',
      destinationPostalCode: '10001',
      weight: 5.0,
      weightUnitOfMeasure: 'LBS'
    };

    const response = await client.getTransitTimes(request);
    expect(response.emsResponse).toBeDefined();
    expect(mockGetToken).toHaveBeenCalled();
  });

  it('should find fastest service', async () => {
    const request: TimeInTransitRequest = {
      originCountryCode: 'US',
      originPostalCode: '94105',
      destinationCountryCode: 'US',
      destinationPostalCode: '10001'
    };

    const fastestService = await client.getFastestService(request);
    expect(fastestService).toBeDefined();
    expect(fastestService?.businessTransitDays).toBeDefined();
  });

  it('should handle errors gracefully', async () => {
    mockGetToken.mockRejectedValue(new Error('Token expired'));
    
    await expect(
      client.getTransitTimes({
        originCountryCode: 'US',
        destinationCountryCode: 'US'
      })
    ).rejects.toThrow('Token expired');
  });
});
```

---

Happy shipping with accurate transit times!
