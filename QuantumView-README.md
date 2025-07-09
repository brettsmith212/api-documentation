# UPS QuantumView API
Comprehensive README for Node.js + TypeScript Integrators

---

## 1. What Is the QuantumView API?

The **QuantumView API** is a powerful suite of services that provides detailed tracking and shipment information for UPS packages. It gives you and your customers comprehensive visibility into UPS shipments through subscription-based data feeds.

Key features:
* **Subscription-based data feeds** for inbound and outbound shipments
* **Real-time shipment events** including origin, delivery, and exception events
* **Detailed manifest information** with shipper, consignee, and package details
* **Bookmark-based pagination** for large datasets
* **Reference number tracking** for easier shipment identification
* **Comprehensive delivery information** including signatures and locations

Common use cases:
* Automated shipment status notifications for customers
* Supply chain visibility dashboards
* Exception handling and proactive customer service
* Inventory management and receiving automation
* Business intelligence and analytics on shipping patterns

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/quantumview/v3/events` | Get quantum view events for subscriptions |
| `POST` | `/quantumview/v1/events` | Legacy endpoint (deprecated) |

### 2.1 Required Headers

| Name | Notes |
|------|-------|
| `Authorization` | OAuth 2.0 Bearer token |
| `Content-Type` | `application/json` |

### 2.2 Path Parameters

| Param | Type | Example | Description |
|-------|------|---------|-------------|
| `version` | `'v3'` | `v3` | API version (v3 recommended) |
| `deprecatedVersion` | `'v1'` | `v1` | Legacy version (avoid) |

### 2.3 Request Body Structure

```json
{
  "QuantumViewRequest": {
    "Request": {
      "RequestAction": "QVEvents",
      "TransactionReference": {
        "CustomerContext": "Your reference"
      }
    },
    "SubscriptionRequest": [
      {
        "Name": "OutboundXML",
        "FileName": ["220104_140019001"],
        "DateTimeRange": {
          "BeginDateTime": "20240101120000",
          "EndDateTime": "20240101130000"
        }
      }
    ],
    "Bookmark": "WE9MVFFBMQ=="
  }
}
```

---

## 3. TypeScript Interfaces

```ts
// OAuth Token
export interface OAuthToken {
  access_token: string;
  token_type: 'Bearer';
  expires_in: number;
}

// Request Interfaces
export interface QuantumViewRequest {
  Request: {
    RequestAction: 'QVEvents';
    TransactionReference?: {
      CustomerContext?: string;
    };
  };
  SubscriptionRequest?: SubscriptionRequest[];
  Bookmark?: string;
}

export interface SubscriptionRequest {
  Name?: string;
  FileName?: string[];
  DateTimeRange?: {
    BeginDateTime?: string; // YYYYMMDDHHMMSS
    EndDateTime?: string;   // YYYYMMDDHHMMSS
  };
}

export interface QuantumViewRequestWrapper {
  QuantumViewRequest: QuantumViewRequest;
}

// Response Interfaces
export interface QuantumViewResponse {
  Response: {
    TransactionReference?: {
      CustomerContext?: string;
      XpciVersion?: string;
      ToolVersion?: string;
    };
    ResponseStatusCode: '1' | '0';
    ResponseStatusDescription?: string;
    Error?: ResponseError[];
  };
  QuantumViewEvents: {
    SubscriberID: string;
    SubscriptionEvents: SubscriptionEvent[];
  };
  Bookmark?: string;
}

export interface QuantumViewResponseWrapper {
  QuantumViewResponse: QuantumViewResponse;
}

export interface SubscriptionEvent {
  Name?: string;
  Number?: string;
  SubscriptionStatus: {
    Code: 'UN' | 'AT' | 'P' | 'A' | 'I' | 'S';
    Description?: string;
  };
  DateRange?: {
    BeginDate: string; // MM-DD-YYYY-HH-MM
    EndDate?: string;  // MM-DD-YYYY-HH-MM
  };
  SubscriptionFile?: SubscriptionFile[];
}

export interface SubscriptionFile {
  FileName: string;
  StatusType: {
    Code: 'R' | 'U'; // Read or Unread
    Description: string;
  };
  Manifest?: Manifest[];
  Origin?: Origin[];
  Exception?: Exception[];
  Delivery?: Delivery[];
  Generic?: Generic[];
}

// Detailed Event Types
export interface Manifest {
  Shipper: {
    Name: string;
    AttentionName?: string;
    ShipperNumber?: string;
    PhoneNumber?: string;
    EMailAddress?: string;
    Address: Address;
  };
  ShipTo: {
    CompanyName?: string;
    AttentionName?: string;
    PhoneNumber?: string;
    Address: Address;
  };
  Service: {
    Code: string;
    Description?: string;
  };
  Package: Package[];
  ReferenceNumber?: ReferenceNumber[];
  PickupDate?: string;
  ScheduledDeliveryDate?: string;
  ScheduledDeliveryTime?: string;
  ConsigneeBillIndicator?: string;
  CollectBillIndicator?: string;
}

export interface Package {
  TrackingNumber?: string;
  Description?: string;
  PackageWeight?: {
    Weight: string;
  };
  Dimensions?: {
    Length: string;
    Width: string;
    Height: string;
  };
  ReferenceNumber?: ReferenceNumber[];
  Activity?: Activity[];
}

export interface Activity {
  Date: string; // YYYYMMDD
  Time: string; // HHMMSS
}

export interface Origin {
  TrackingNumber: string;
  ShipperNumber: string;
  Date: string; // YYYYMMDD
  Time: string; // HHMMSS
  PackageReferenceNumber?: ReferenceNumber[];
  ShipmentReferenceNumber?: ReferenceNumber[];
  ScheduledDeliveryDate?: string;
  ScheduledDeliveryTime?: string;
  ActivityLocation?: ActivityLocation;
}

export interface Exception {
  TrackingNumber: string;
  ShipperNumber: string;
  Date: string; // YYYYMMDD
  Time: string; // HHMMSS
  StatusCode?: string;
  StatusDescription?: string;
  ReasonCode?: string;
  ReasonDescription?: string;
  UpdatedAddress?: UpdatedAddress;
  RescheduledDeliveryDate?: string;
  RescheduledDeliveryTime?: string;
  ActivityLocation?: ActivityLocation;
}

export interface Delivery {
  TrackingNumber: string;
  ShipperNumber: string;
  Date: string; // YYYYMMDD
  Time: string; // HHMMSS
  DriverRelease?: string;
  DeliveryLocation?: DeliveryLocation;
  ActivityLocation?: ActivityLocation;
}

export interface Generic {
  TrackingNumber: string;
  ShipperNumber?: string;
  ActivityType: 'VM' | 'UR' | 'IR' | 'TC' | 'PS' | 'FN' | 'DS' | 'AG' | 'RE' | 'RP' | 'UD' | 'OD' | 'SD' | 'FM' | 'PT' | 'PC';
  Activity?: Activity;
  RescheduledDeliveryDate?: string;
}

// Supporting Types
export interface Address {
  AddressLine1?: string;
  AddressLine2?: string;
  AddressLine3?: string;
  City?: string;
  StateProvinceCode?: string;
  PostalCode?: string;
  CountryCode?: string;
  ResidentialAddressIndicator?: string;
}

export interface ReferenceNumber {
  Code?: string;
  Value: string;
}

export interface ActivityLocation {
  AddressArtifactFormat: {
    PoliticalDivision2?: string; // City
    PoliticalDivision1?: string; // State
    CountryCode?: string;
  };
}

export interface DeliveryLocation {
  AddressArtifactFormat: {
    ConsigneeName?: string;
    PoliticalDivision2?: string;
    PoliticalDivision1?: string;
    CountryCode?: string;
  };
  SignedForByName?: string;
  Code?: string;
  Description?: string;
}

export interface UpdatedAddress {
  ConsigneeName?: string;
  PoliticalDivision2?: string;
  PoliticalDivision1?: string;
  CountryCode?: string;
  PostcodePrimaryLow?: string;
}

export interface ResponseError {
  ErrorSeverity?: string;
  ErrorCode?: string;
  ErrorDescription?: string;
  ErrorLocation?: {
    ErrorLocationElementName?: string;
    ErrorLocationAttributeName?: string;
  }[];
}

// Error Response
export interface QuantumViewError {
  code: string;
  message: string;
}

export interface QuantumViewErrorResponse {
  response: {
    errors: QuantumViewError[];
  };
}
```

---

## 4. Authentication (OAuth 2.0)

```
POST https://wwwcie.ups.com/security/v1/oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(<client_id>:<client_secret>)

grant_type=client_credentials
```

* **Test Environment**: `https://wwwcie.ups.com`
* **Production**: `https://onlinetools.ups.com`
* **Token Lifetime**: ~1 hour (cache and reuse)

---

## 5. Production-Ready Node.js Implementation

### 5.1 Install Dependencies

```bash
npm i axios uuid
npm i -D @types/node @types/uuid
```

### 5.2 `upsQuantumViewClient.ts`

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import { v4 as uuidv4 } from 'uuid';
import {
  OAuthToken,
  QuantumViewRequestWrapper,
  QuantumViewResponseWrapper,
  QuantumViewErrorResponse
} from './types/ups-quantumview';

export interface UpsQuantumViewClientConfig {
  clientId: string;
  clientSecret: string;
  baseURL?: string;
  timeout?: number;
}

export class UpsQuantumViewClient {
  private axios: AxiosInstance;
  private token?: OAuthToken;
  private tokenFetchedAt = 0;

  constructor(private readonly config: UpsQuantumViewClientConfig) {
    this.axios = axios.create({
      baseURL: config.baseURL ?? 'https://wwwcie.ups.com/api',
      timeout: config.timeout ?? 30_000,
      headers: {
        'Content-Type': 'application/json'
      }
    });
  }

  /**
   * Get OAuth token with caching
   */
  private async getAccessToken(): Promise<string> {
    const now = Date.now();
    
    if (this.token && now - this.tokenFetchedAt < (this.token.expires_in - 60) * 1000) {
      return this.token.access_token;
    }

    const credentials = Buffer.from(
      `${this.config.clientId}:${this.config.clientSecret}`
    ).toString('base64');

    try {
      const { data } = await axios.post<OAuthToken>(
        `${this.config.baseURL ?? 'https://wwwcie.ups.com'}/security/v1/oauth/token`,
        'grant_type=client_credentials',
        {
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Authorization': `Basic ${credentials}`
          }
        }
      );

      this.token = data;
      this.tokenFetchedAt = now;
      return data.access_token;
    } catch (error) {
      throw new Error(`Failed to get access token: ${error}`);
    }
  }

  /**
   * Get quantum view events
   */
  async getQuantumViewEvents(
    request: QuantumViewRequestWrapper,
    version: 'v3' | 'v1' = 'v3'
  ): Promise<QuantumViewResponseWrapper> {
    const accessToken = await this.getAccessToken();

    try {
      const { data } = await this.axios.post<QuantumViewResponseWrapper>(
        `/quantumview/${version}/events`,
        request,
        {
          headers: {
            'Authorization': `Bearer ${accessToken}`
          }
        }
      );

      return data;
    } catch (error) {
      this.handleQuantumViewError(error);
      throw error;
    }
  }

  /**
   * Get events by subscription name
   */
  async getEventsBySubscription(
    subscriptionName: string,
    dateRange?: {
      beginDateTime: string;
      endDateTime: string;
    }
  ): Promise<QuantumViewResponseWrapper> {
    const request: QuantumViewRequestWrapper = {
      QuantumViewRequest: {
        Request: {
          RequestAction: 'QVEvents',
          TransactionReference: {
            CustomerContext: `Subscription: ${subscriptionName}`
          }
        },
        SubscriptionRequest: [
          {
            Name: subscriptionName,
            ...(dateRange && { DateTimeRange: dateRange })
          }
        ]
      }
    };

    return this.getQuantumViewEvents(request);
  }

  /**
   * Get events by file name
   */
  async getEventsByFileName(
    fileName: string,
    subscriptionName?: string
  ): Promise<QuantumViewResponseWrapper> {
    const request: QuantumViewRequestWrapper = {
      QuantumViewRequest: {
        Request: {
          RequestAction: 'QVEvents',
          TransactionReference: {
            CustomerContext: `File: ${fileName}`
          }
        },
        SubscriptionRequest: [
          {
            FileName: [fileName],
            ...(subscriptionName && { Name: subscriptionName })
          }
        ]
      }
    };

    return this.getQuantumViewEvents(request);
  }

  /**
   * Get paginated events using bookmark
   */
  async getEventsByBookmark(
    bookmark: string,
    subscriptionName?: string
  ): Promise<QuantumViewResponseWrapper> {
    const request: QuantumViewRequestWrapper = {
      QuantumViewRequest: {
        Request: {
          RequestAction: 'QVEvents',
          TransactionReference: {
            CustomerContext: 'Bookmark pagination'
          }
        },
        Bookmark: bookmark,
        ...(subscriptionName && {
          SubscriptionRequest: [{ Name: subscriptionName }]
        })
      }
    };

    return this.getQuantumViewEvents(request);
  }

  /**
   * Get all events with automatic pagination
   */
  async getAllEvents(
    subscriptionName: string,
    dateRange?: { beginDateTime: string; endDateTime: string }
  ): Promise<QuantumViewResponseWrapper[]> {
    const results: QuantumViewResponseWrapper[] = [];
    let bookmark: string | undefined;

    do {
      const request: QuantumViewRequestWrapper = {
        QuantumViewRequest: {
          Request: {
            RequestAction: 'QVEvents',
            TransactionReference: {
              CustomerContext: `All events: ${subscriptionName}`
            }
          },
          SubscriptionRequest: [
            {
              Name: subscriptionName,
              ...(dateRange && { DateTimeRange: dateRange })
            }
          ],
          ...(bookmark && { Bookmark: bookmark })
        }
      };

      const response = await this.getQuantumViewEvents(request);
      results.push(response);
      
      bookmark = response.QuantumViewResponse.Bookmark;
    } while (bookmark);

    return results;
  }

  /**
   * Process quantum view events with callback
   */
  async processEvents(
    subscriptionName: string,
    processor: (events: QuantumViewResponseWrapper) => Promise<void>,
    dateRange?: { beginDateTime: string; endDateTime: string }
  ): Promise<void> {
    let bookmark: string | undefined;

    do {
      const request: QuantumViewRequestWrapper = {
        QuantumViewRequest: {
          Request: {
            RequestAction: 'QVEvents',
            TransactionReference: {
              CustomerContext: `Process events: ${subscriptionName}`
            }
          },
          SubscriptionRequest: [
            {
              Name: subscriptionName,
              ...(dateRange && { DateTimeRange: dateRange })
            }
          ],
          ...(bookmark && { Bookmark: bookmark })
        }
      };

      const response = await this.getQuantumViewEvents(request);
      await processor(response);
      
      bookmark = response.QuantumViewResponse.Bookmark;
    } while (bookmark);
  }

  private handleQuantumViewError(error: unknown): void {
    if (axios.isAxiosError(error)) {
      const e = error as AxiosError<QuantumViewErrorResponse>;
      const status = e.response?.status;
      const errorData = e.response?.data?.response?.errors;

      switch (status) {
        case 400:
          throw new Error(`Invalid request: ${errorData?.map(err => err.message).join(', ')}`);
        case 401:
          throw new Error('Unauthorized - check your credentials');
        case 403:
          throw new Error('Forbidden - insufficient permissions');
        case 429:
          throw new Error('Rate limit exceeded - please retry later');
        case 500:
          throw new Error('UPS service temporarily unavailable');
        default:
          const message = errorData?.map(err => err.message).join(', ') || 'Unknown error';
          throw new Error(`QuantumView API error: ${message}`);
      }
    }

    throw new Error(`QuantumView request failed: ${error}`);
  }
}

// Helper function to format dates for API
export function formatDateTime(date: Date): string {
  return date.toISOString().replace(/[-:T]/g, '').substring(0, 14);
}

// Helper function to parse quantum view dates
export function parseQuantumViewDate(dateStr: string): Date {
  // Handle YYYYMMDD format
  if (dateStr.length === 8) {
    const year = parseInt(dateStr.substring(0, 4));
    const month = parseInt(dateStr.substring(4, 6)) - 1;
    const day = parseInt(dateStr.substring(6, 8));
    return new Date(year, month, day);
  }
  
  // Handle YYYYMMDDHHMMSS format
  if (dateStr.length === 14) {
    const year = parseInt(dateStr.substring(0, 4));
    const month = parseInt(dateStr.substring(4, 6)) - 1;
    const day = parseInt(dateStr.substring(6, 8));
    const hour = parseInt(dateStr.substring(8, 10));
    const minute = parseInt(dateStr.substring(10, 12));
    const second = parseInt(dateStr.substring(12, 14));
    return new Date(year, month, day, hour, minute, second);
  }
  
  throw new Error(`Invalid date format: ${dateStr}`);
}
```

---

## 6. Usage Examples

### 6.1 Basic Setup

```ts
import { UpsQuantumViewClient, formatDateTime } from './upsQuantumViewClient';

const client = new UpsQuantumViewClient({
  clientId: process.env.UPS_CLIENT_ID!,
  clientSecret: process.env.UPS_CLIENT_SECRET!,
  baseURL: process.env.NODE_ENV === 'production' 
    ? 'https://onlinetools.ups.com/api'
    : 'https://wwwcie.ups.com/api'
});
```

### 6.2 Get Events by Subscription

```ts
async function getRecentEvents() {
  try {
    const now = new Date();
    const yesterday = new Date(now.getTime() - 24 * 60 * 60 * 1000);
    
    const response = await client.getEventsBySubscription('OutboundXML', {
      beginDateTime: formatDateTime(yesterday),
      endDateTime: formatDateTime(now)
    });
    
    const events = response.QuantumViewResponse.QuantumViewEvents;
    console.log(`Found ${events.SubscriptionEvents.length} subscription events`);
    
    for (const event of events.SubscriptionEvents) {
      console.log(`Subscription: ${event.Name}, Status: ${event.SubscriptionStatus.Description}`);
      
      if (event.SubscriptionFile) {
        for (const file of event.SubscriptionFile) {
          console.log(`  File: ${file.FileName}, Status: ${file.StatusType.Description}`);
          
          // Process manifest data
          if (file.Manifest) {
            file.Manifest.forEach(manifest => {
              console.log(`    Shipment from ${manifest.Shipper.Name} to ${manifest.ShipTo.CompanyName}`);
            });
          }
        }
      }
    }
  } catch (error) {
    console.error('Error getting events:', error.message);
  }
}
```

### 6.3 Process All Events with Pagination

```ts
async function processAllOutboundEvents() {
  try {
    await client.processEvents('OutboundXML', async (response) => {
      const events = response.QuantumViewResponse.QuantumViewEvents;
      
      for (const event of events.SubscriptionEvents) {
        if (event.SubscriptionFile) {
          for (const file of event.SubscriptionFile) {
            // Process origin events
            if (file.Origin) {
              for (const origin of file.Origin) {
                console.log(`Package ${origin.TrackingNumber} picked up on ${origin.Date}`);
                await updatePackageStatus(origin.TrackingNumber, 'IN_TRANSIT');
              }
            }
            
            // Process delivery events
            if (file.Delivery) {
              for (const delivery of file.Delivery) {
                console.log(`Package ${delivery.TrackingNumber} delivered on ${delivery.Date}`);
                await updatePackageStatus(delivery.TrackingNumber, 'DELIVERED');
              }
            }
            
            // Process exceptions
            if (file.Exception) {
              for (const exception of file.Exception) {
                console.log(`Exception for ${exception.TrackingNumber}: ${exception.StatusDescription}`);
                await handleException(exception);
              }
            }
          }
        }
      }
    });
  } catch (error) {
    console.error('Error processing events:', error.message);
  }
}
```

### 6.4 Monitor Specific Package Activities

```ts
async function monitorPackageActivities(trackingNumbers: string[]) {
  try {
    const response = await client.getEventsBySubscription('OutboundXML');
    const events = response.QuantumViewResponse.QuantumViewEvents;
    
    for (const event of events.SubscriptionEvents) {
      if (event.SubscriptionFile) {
        for (const file of event.SubscriptionFile) {
          // Check all event types for our tracking numbers
          const allEvents = [
            ...(file.Origin || []),
            ...(file.Delivery || []),
            ...(file.Exception || []),
            ...(file.Generic || [])
          ];
          
          for (const eventItem of allEvents) {
            if (trackingNumbers.includes(eventItem.TrackingNumber)) {
              console.log(`Update for ${eventItem.TrackingNumber}:`);
              
              if ('Date' in eventItem && 'Time' in eventItem) {
                console.log(`  Date: ${eventItem.Date}, Time: ${eventItem.Time}`);
              }
              
              if ('ActivityType' in eventItem) {
                console.log(`  Activity: ${eventItem.ActivityType}`);
              }
              
              if ('StatusDescription' in eventItem) {
                console.log(`  Status: ${eventItem.StatusDescription}`);
              }
            }
          }
        }
      }
    }
  } catch (error) {
    console.error('Error monitoring activities:', error.message);
  }
}
```

---

## 7. Error Handling

```ts
function handleQuantumViewError(error: unknown) {
  if (error instanceof Error) {
    switch (true) {
      case error.message.includes('Invalid request'):
        console.log('Check your request parameters and subscription configuration');
        break;
      case error.message.includes('Unauthorized'):
        console.log('Check your OAuth credentials and permissions');
        break;
      case error.message.includes('Rate limit'):
        console.log('Implement exponential backoff - too many requests');
        break;
      case error.message.includes('temporarily unavailable'):
        console.log('UPS service down - retry later');
        break;
      default:
        console.error('QuantumView error:', error.message);
    }
  }
}
```

---

## 8. Best Practices for Quantum View Data

### 8.1 Subscription Management
1. **Use meaningful subscription names** - helps with debugging and monitoring
2. **Set appropriate date ranges** - avoid overly broad queries
3. **Handle pagination properly** - use bookmarks for large datasets
4. **Cache subscription configurations** - avoid repeated lookups

### 8.2 Data Processing
1. **Process events in chronological order** - use Date/Time fields for sorting
2. **Handle duplicate events** - use tracking numbers and timestamps as keys
3. **Implement idempotent processing** - same event should produce same result
4. **Store raw event data** - useful for debugging and reprocessing

### 8.3 Performance Optimization
1. **Use bookmarks for pagination** - more efficient than date-based queries
2. **Batch process events** - don't make individual API calls for each package
3. **Implement rate limiting** - respect UPS API limits
4. **Cache frequently accessed data** - reduce API calls

### 8.4 Error Recovery
1. **Implement retry logic** - with exponential backoff for failures
2. **Handle partial failures** - process successful events even if some fail
3. **Monitor subscription health** - check subscription status regularly
4. **Log correlation IDs** - use CustomerContext for debugging

### 8.5 Data Integrity
1. **Validate event sequences** - ensure logical order of events
2. **Cross-reference with tracking** - verify against Tracking API data
3. **Handle time zone differences** - events may be in different time zones
4. **Implement data reconciliation** - periodic validation against source systems

---

## 9. Testing

```ts
// Example test setup
describe('UPS QuantumView Client', () => {
  let client: UpsQuantumViewClient;
  
  beforeEach(() => {
    client = new UpsQuantumViewClient({
      clientId: 'test-client-id',
      clientSecret: 'test-client-secret',
      baseURL: 'https://wwwcie.ups.com/api'
    });
  });
  
  it('should get events by subscription', async () => {
    const response = await client.getEventsBySubscription('TestSubscription');
    expect(response.QuantumViewResponse.Response.ResponseStatusCode).toBe('1');
  });
  
  it('should handle pagination with bookmarks', async () => {
    const firstPage = await client.getEventsBySubscription('TestSubscription');
    
    if (firstPage.QuantumViewResponse.Bookmark) {
      const secondPage = await client.getEventsByBookmark(
        firstPage.QuantumViewResponse.Bookmark
      );
      expect(secondPage.QuantumViewResponse.Response.ResponseStatusCode).toBe('1');
    }
  });
});
```

---

## 10. Integration Patterns

### 10.1 Real-time Event Processing

```ts
class QuantumViewProcessor {
  private client: UpsQuantumViewClient;
  private lastBookmark?: string;
  
  constructor(client: UpsQuantumViewClient) {
    this.client = client;
  }
  
  async processNewEvents(subscriptionName: string): Promise<void> {
    try {
      const response = this.lastBookmark
        ? await this.client.getEventsByBookmark(this.lastBookmark, subscriptionName)
        : await this.client.getEventsBySubscription(subscriptionName);
      
      await this.processEventData(response);
      
      // Update bookmark for next call
      this.lastBookmark = response.QuantumViewResponse.Bookmark;
    } catch (error) {
      console.error('Error processing events:', error);
    }
  }
  
  private async processEventData(response: QuantumViewResponseWrapper): Promise<void> {
    // Process events based on type
    const events = response.QuantumViewResponse.QuantumViewEvents;
    
    for (const event of events.SubscriptionEvents) {
      if (event.SubscriptionFile) {
        for (const file of event.SubscriptionFile) {
          await this.processManifestEvents(file.Manifest || []);
          await this.processOriginEvents(file.Origin || []);
          await this.processDeliveryEvents(file.Delivery || []);
          await this.processExceptionEvents(file.Exception || []);
        }
      }
    }
  }
}
```

---

Happy quantum viewing and shipment tracking!
