# UPS Tracking API
Comprehensive README for Node.js + TypeScript Integrators  

---

## 1. What Is the UPS Tracking API?
The **UPS Tracking API** retrieves real-time tracking information for UPS shipments including Small Package, Mail Innovations, and UPS Freight. It provides detailed package status, activity history, delivery information, and proof of delivery.

Key features:
* **Real-time tracking** for multiple shipment types
* **Activity history** with timestamps and locations
* **Proof of delivery** with signature capture
* **Delivery notifications** and estimated delivery dates
* **Reference number tracking** for easier shipment lookup
* **120-day data retention** period

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/track/v1/details/{inquiryNumber}` | Track by tracking number |
| `GET` | `/track/v1/reference/details/{referenceNumber}` | Track by reference number |

### 2.1 Path Parameters
* `inquiryNumber` - UPS tracking number (7-34 characters)
* `referenceNumber` - Reference number associated with shipment

### 2.2 Query Parameters

#### For tracking number endpoint:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `locale` | string | `en_US` | Language/country code (e.g., `en_US`, `es_ES`) |
| `returnSignature` | string | `false` | Return delivery signature image |
| `returnMilestones` | string | `false` | Return milestone information |
| `returnPOD` | string | `false` | Return Proof of Delivery |

#### For reference number endpoint:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `locale` | string | `en_US` | Language/country code |
| `fromPickUpDate` | string | `currentDate-14` | Start date for search (YYYYMMDD) |
| `toPickUpDate` | string | `currentDate` | End date for search (YYYYMMDD) |
| `refNumType` | string | `SmallPackage` | Type: `SmallPackage` or `fgv` |

### 2.3 Required Headers
* `Authorization: Bearer {access_token}`
* `transId: {unique_transaction_id}`
* `transactionSrc: {source_application_name}`

---

## 3. TypeScript Interfaces

```ts
// Request Configuration
export interface TrackingRequestConfig {
  locale?: string;
  returnSignature?: boolean;
  returnMilestones?: boolean;
  returnPOD?: boolean;
}

export interface ReferenceTrackingConfig {
  locale?: string;
  fromPickUpDate?: string;
  toPickUpDate?: string;
  refNumType?: 'SmallPackage' | 'fgv';
}

// Response Types
export interface TrackingResponse {
  trackResponse: {
    shipment: Shipment[];
  };
}

export interface Shipment {
  inquiryNumber: string;
  package: Package[];
  userRelation?: string[];
  warnings?: Warning[];
}

export interface Package {
  trackingNumber: string;
  packageCount: number;
  currentStatus: Status;
  statusCode: string;
  statusDescription: string;
  activity: Activity[];
  deliveryDate?: DeliveryDate[];
  deliveryTime?: DeliveryTime;
  deliveryInformation?: DeliveryInformation;
  packageAddress: PackageAddress[];
  service: Service;
  weight?: Weight;
  dimension?: Dimension;
  referenceNumber?: ReferenceNumber[];
  alternateTrackingNumber?: AlternateTrackingNumber[];
  additionalServices?: string[];
  additionalAttributes?: string[];
  milestones?: Milestone[];
  paymentInformation?: PaymentInformation[];
  accessPointInformation?: AccessPointInformation;
  isSmartPackage?: boolean;
  ucixStatus?: string;
  suppressionIndicators?: string[];
}

export interface Activity {
  date: string; // YYYYMMDD
  time: string; // HHMMSS
  gmtDate?: string;
  gmtTime?: string;
  gmtOffset?: string;
  status: Status;
  location: Location;
}

export interface Status {
  code: string;
  description: string;
  simplifiedTextDescription?: string;
  statusCode?: string;
  type?: string;
}

export interface Location {
  address: Address;
  slic?: string;
}

export interface Address {
  addressLine1?: string;
  addressLine2?: string;
  addressLine3?: string;
  city?: string;
  stateProvince?: string;
  postalCode?: string;
  country?: string;
  countryCode?: string;
}

export interface DeliveryDate {
  date: string; // YYYYMMDD
  type: 'SDD' | 'RDD' | 'DEL'; // Scheduled/Rescheduled/Delivered
}

export interface DeliveryTime {
  startTime?: string; // HHMMSS
  endTime?: string; // HHMMSS
  type: 'EOD' | 'CMT' | 'EDW' | 'CDW' | 'IDW' | 'DEL';
}

export interface DeliveryInformation {
  location?: string;
  receivedBy?: string;
  signature?: {
    image: string; // Base64 encoded
  };
  deliveryPhoto?: {
    photo: string;
    photoCaptureInd: 'Y' | 'N';
    photoDispositionCode: 'V' | 'N' | 'U';
    isNonPostalCodeCountry: boolean;
  };
  pod?: {
    content: string; // Base64 encoded
  };
}

export interface PackageAddress {
  type: 'ORIGIN' | 'DESTINATION';
  name?: string;
  attentionName?: string;
  address: Address;
}

export interface Service {
  code: string;
  description: string;
  levelCode?: string;
}

export interface Weight {
  weight: string;
  unitOfMeasurement: 'LBS' | 'KGS';
}

export interface Dimension {
  length?: string;
  width?: string;
  height?: string;
  unitOfDimension?: string;
}

export interface ReferenceNumber {
  type: string;
  number: string;
}

export interface AlternateTrackingNumber {
  type: string;
  number: string;
}

export interface Milestone {
  code: string;
  description: string;
  current: boolean;
  state: string;
  category?: string;
  linkedActivity?: string;
  subMilestone?: {
    category: string;
  };
}

export interface PaymentInformation {
  type: string;
  amount: string;
  currency: string;
  paid: boolean;
  paymentMethod?: string;
  id?: string;
}

export interface AccessPointInformation {
  pickupByDate?: string;
}

export interface Warning {
  code: string;
  message: string;
}

// Error Response
export interface TrackingError {
  code: string;
  message: string;
}

export interface TrackingErrorResponse {
  response: {
    errors: TrackingError[];
  };
}
```

---

## 4. Production-Ready Node.js Implementation

### 4.1 Install Dependencies

```bash
npm i axios uuid
npm i -D @types/node @types/uuid
```

### 4.2 `upsTrackingClient.ts` - Tracking Client

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import { v4 as uuidv4 } from 'uuid';
import {
  TrackingResponse,
  TrackingRequestConfig,
  ReferenceTrackingConfig,
  TrackingErrorResponse
} from './types/ups-tracking';

export interface UpsTrackingClientConfig {
  baseURL?: string;
  timeout?: number;
  transactionSrc?: string;
}

export class UpsTrackingClient {
  private axios: AxiosInstance;
  private transactionSrc: string;

  constructor(
    private getAccessToken: () => Promise<string>,
    config: UpsTrackingClientConfig = {}
  ) {
    this.axios = axios.create({
      baseURL: config.baseURL ?? 'https://wwwcie.ups.com/api',
      timeout: config.timeout ?? 30_000
    });

    this.transactionSrc = config.transactionSrc ?? 'UPS_TRACKING_API';
  }

  /**
   * Track package by tracking number
   */
  async trackByNumber(
    trackingNumber: string,
    config: TrackingRequestConfig = {}
  ): Promise<TrackingResponse> {
    const accessToken = await this.getAccessToken();
    const transId = uuidv4();

    const params = {
      locale: config.locale ?? 'en_US',
      returnSignature: config.returnSignature?.toString() ?? 'false',
      returnMilestones: config.returnMilestones?.toString() ?? 'false',
      returnPOD: config.returnPOD?.toString() ?? 'false'
    };

    try {
      const { data } = await this.axios.get<TrackingResponse>(
        `/track/v1/details/${encodeURIComponent(trackingNumber)}`,
        {
          params,
          headers: {
            'Authorization': `Bearer ${accessToken}`,
            'transId': transId,
            'transactionSrc': this.transactionSrc
          }
        }
      );

      return data;
    } catch (error) {
      this.handleTrackingError(error, trackingNumber);
      throw error;
    }
  }

  /**
   * Track package by reference number
   */
  async trackByReference(
    referenceNumber: string,
    config: ReferenceTrackingConfig = {}
  ): Promise<TrackingResponse> {
    const accessToken = await this.getAccessToken();
    const transId = uuidv4();

    const today = new Date();
    const twoWeeksAgo = new Date(today);
    twoWeeksAgo.setDate(today.getDate() - 14);

    const params = {
      locale: config.locale ?? 'en_US',
      fromPickUpDate: config.fromPickUpDate ?? this.formatDate(twoWeeksAgo),
      toPickUpDate: config.toPickUpDate ?? this.formatDate(today),
      refNumType: config.refNumType ?? 'SmallPackage'
    };

    try {
      const { data } = await this.axios.get<TrackingResponse>(
        `/track/v1/reference/details/${encodeURIComponent(referenceNumber)}`,
        {
          params,
          headers: {
            'Authorization': `Bearer ${accessToken}`,
            'transId': transId,
            'transactionSrc': this.transactionSrc
          }
        }
      );

      return data;
    } catch (error) {
      this.handleTrackingError(error, referenceNumber);
      throw error;
    }
  }

  /**
   * Get current status of a package
   */
  async getPackageStatus(trackingNumber: string): Promise<string> {
    const response = await this.trackByNumber(trackingNumber);
    const package_ = response.trackResponse.shipment[0]?.package[0];
    
    if (!package_) {
      throw new Error('Package not found');
    }

    return package_.currentStatus.description;
  }

  /**
   * Get delivery information if package is delivered
   */
  async getDeliveryInfo(trackingNumber: string) {
    const response = await this.trackByNumber(trackingNumber, {
      returnSignature: true,
      returnPOD: true
    });
    
    const package_ = response.trackResponse.shipment[0]?.package[0];
    
    if (!package_) {
      throw new Error('Package not found');
    }

    return {
      isDelivered: package_.currentStatus.code === 'D',
      deliveryInformation: package_.deliveryInformation,
      deliveryDate: package_.deliveryDate?.find(d => d.type === 'DEL')?.date,
      deliveryTime: package_.deliveryTime?.type === 'DEL' ? package_.deliveryTime.endTime : undefined
    };
  }

  /**
   * Get activity history for a package
   */
  async getActivityHistory(trackingNumber: string) {
    const response = await this.trackByNumber(trackingNumber);
    const package_ = response.trackResponse.shipment[0]?.package[0];
    
    if (!package_) {
      throw new Error('Package not found');
    }

    return package_.activity.map(activity => ({
      date: activity.date,
      time: activity.time,
      status: activity.status.description,
      location: {
        city: activity.location.address.city,
        state: activity.location.address.stateProvince,
        country: activity.location.address.country
      }
    }));
  }

  /**
   * Track multiple packages
   */
  async trackMultiple(trackingNumbers: string[]): Promise<Map<string, TrackingResponse | Error>> {
    const results = new Map<string, TrackingResponse | Error>();
    
    // Process in batches to avoid rate limiting
    const batchSize = 5;
    for (let i = 0; i < trackingNumbers.length; i += batchSize) {
      const batch = trackingNumbers.slice(i, i + batchSize);
      
      const promises = batch.map(async (trackingNumber) => {
        try {
          const result = await this.trackByNumber(trackingNumber);
          results.set(trackingNumber, result);
        } catch (error) {
          results.set(trackingNumber, error as Error);
        }
      });
      
      await Promise.all(promises);
      
      // Small delay between batches
      if (i + batchSize < trackingNumbers.length) {
        await new Promise(resolve => setTimeout(resolve, 100));
      }
    }
    
    return results;
  }

  private formatDate(date: Date): string {
    return date.toISOString().slice(0, 10).replace(/-/g, '');
  }

  private handleTrackingError(error: unknown, trackingNumber: string): void {
    if (axios.isAxiosError(error)) {
      const e = error as AxiosError<TrackingErrorResponse>;
      const status = e.response?.status;
      const errorData = e.response?.data?.response?.errors;

      switch (status) {
        case 400:
          throw new Error(`Invalid tracking number: ${trackingNumber}`);
        case 404:
          throw new Error(`Tracking information not found for: ${trackingNumber}`);
        case 403:
          throw new Error('Access forbidden - check your credentials');
        case 429:
          throw new Error('Rate limit exceeded - please retry later');
        case 500:
          throw new Error('UPS service temporarily unavailable');
        default:
          const message = errorData?.map(err => err.message).join(', ') || 'Unknown error';
          throw new Error(`Tracking failed: ${message}`);
      }
    }

    throw new Error(`Tracking failed for ${trackingNumber}: ${error}`);
  }
}
```

### 4.3 Usage Examples

```ts
import { UpsTrackingClient } from './upsTrackingClient';
import { UpsOAuthClient } from './upsOAuthClient';

// Setup OAuth client for token management
const oauthClient = new UpsOAuthClient({
  clientId: process.env.UPS_CLIENT_ID!,
  clientSecret: process.env.UPS_CLIENT_SECRET!,
  baseURL: process.env.NODE_ENV === 'production' 
    ? 'https://onlinetools.ups.com'
    : 'https://wwwcie.ups.com'
});

// Create tracking client
const trackingClient = new UpsTrackingClient(
  () => oauthClient.getAccessToken(),
  {
    baseURL: process.env.NODE_ENV === 'production' 
      ? 'https://onlinetools.ups.com/api'
      : 'https://wwwcie.ups.com/api',
    transactionSrc: 'MyShippingApp'
  }
);

// Basic tracking
async function trackPackage() {
  try {
    const response = await trackingClient.trackByNumber('1Z999AA1234567890');
    const package_ = response.trackResponse.shipment[0].package[0];
    
    console.log('Current Status:', package_.currentStatus.description);
    console.log('Last Activity:', package_.activity[0].status.description);
  } catch (error) {
    console.error('Tracking failed:', error.message);
  }
}

// Get delivery status
async function checkDelivery() {
  try {
    const deliveryInfo = await trackingClient.getDeliveryInfo('1Z999AA1234567890');
    
    if (deliveryInfo.isDelivered) {
      console.log('Package delivered on:', deliveryInfo.deliveryDate);
      console.log('Delivered to:', deliveryInfo.deliveryInformation?.location);
      console.log('Received by:', deliveryInfo.deliveryInformation?.receivedBy);
    } else {
      console.log('Package not yet delivered');
    }
  } catch (error) {
    console.error('Delivery check failed:', error.message);
  }
}

// Track by reference number
async function trackByReference() {
  try {
    const response = await trackingClient.trackByReference('REF123456', {
      fromPickUpDate: '20240101',
      toPickUpDate: '20240131'
    });
    
    console.log('Found shipments:', response.trackResponse.shipment.length);
  } catch (error) {
    console.error('Reference tracking failed:', error.message);
  }
}

// Track multiple packages
async function trackMultiplePackages() {
  const trackingNumbers = [
    '1Z999AA1234567890',
    '1Z999AA1234567891',
    '1Z999AA1234567892'
  ];
  
  const results = await trackingClient.trackMultiple(trackingNumbers);
  
  for (const [trackingNumber, result] of results) {
    if (result instanceof Error) {
      console.error(`${trackingNumber}: ${result.message}`);
    } else {
      const package_ = result.trackResponse.shipment[0].package[0];
      console.log(`${trackingNumber}: ${package_.currentStatus.description}`);
    }
  }
}
```

---

## 5. Error Handling

```ts
function handleTrackingError(error: unknown, trackingNumber: string) {
  if (error instanceof Error) {
    switch (true) {
      case error.message.includes('not found'):
        console.log(`Package ${trackingNumber} not found - may be too old or invalid`);
        break;
      case error.message.includes('Rate limit'):
        console.log('Rate limited - implement exponential backoff');
        break;
      case error.message.includes('forbidden'):
        console.log('Check OAuth token and permissions');
        break;
      case error.message.includes('temporarily unavailable'):
        console.log('UPS service down - retry later');
        break;
      default:
        console.error('Tracking error:', error.message);
    }
  }
}
```

---

## 6. Best Practices

1. **Cache frequently accessed data** - Store recent tracking results
2. **Implement rate limiting** - Respect UPS API limits
3. **Handle data retention** - Data older than 120 days is not available
4. **Use batch requests** - For multiple tracking numbers
5. **Implement retry logic** - With exponential backoff for failures
6. **Monitor API usage** - Track requests and errors
7. **Store transaction IDs** - For debugging and support
8. **Validate tracking numbers** - Before making API calls
9. **Handle missing data** - Not all fields are always present
10. **Use appropriate locales** - For international customers

### 6.1 Rate Limiting Example

```ts
class RateLimiter {
  private requests: number[] = [];
  private readonly maxRequests: number;
  private readonly timeWindow: number;

  constructor(maxRequests = 100, timeWindowMs = 60000) {
    this.maxRequests = maxRequests;
    this.timeWindow = timeWindowMs;
  }

  async acquire(): Promise<void> {
    const now = Date.now();
    
    // Remove old requests
    this.requests = this.requests.filter(time => now - time < this.timeWindow);
    
    if (this.requests.length >= this.maxRequests) {
      const oldestRequest = Math.min(...this.requests);
      const waitTime = this.timeWindow - (now - oldestRequest);
      await new Promise(resolve => setTimeout(resolve, waitTime));
      return this.acquire();
    }
    
    this.requests.push(now);
  }
}
```

---

## 7. Testing

```ts
// Example test
describe('UPS Tracking Client', () => {
  let client: UpsTrackingClient;
  let mockGetToken: jest.Mock;

  beforeEach(() => {
    mockGetToken = jest.fn().mockResolvedValue('mock-token');
    client = new UpsTrackingClient(mockGetToken);
  });

  it('should track package by number', async () => {
    const response = await client.trackByNumber('1Z999AA1234567890');
    expect(response.trackResponse.shipment).toBeDefined();
    expect(mockGetToken).toHaveBeenCalled();
  });

  it('should handle tracking errors', async () => {
    mockGetToken.mockRejectedValue(new Error('Token expired'));
    
    await expect(
      client.trackByNumber('1Z999AA1234567890')
    ).rejects.toThrow('Token expired');
  });
});
```

---

Happy shipping!
