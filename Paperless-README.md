# UPS Paperless Document API  
Comprehensive README for Node.js + TypeScript Integrators  

---

## 1. What Is the Paperless Document API?

The **Paperless Document API** streamlines international shipping by allowing you to upload, manage, and process customs and trade documents digitally. This eliminates the need for physical paperwork and enables faster customs clearance.

**Key capabilities:**
* Upload trade documents (commercial invoices, certificates of origin, export licenses, etc.)
* Push documents to UPS image repository for shipment processing
* Delete uploaded documents when no longer needed
* Support for 13 different document types per shipment
* Base64 encoded file uploads up to 10MB (1MB in test environment)

It's a REST/JSON service secured with OAuth 2 **Client-Credentials** tokens.  
Current stable version path is `v2`.

**Typical use-cases:**
* E-commerce platforms handling international shipments
* Freight forwarders managing customs documentation
* Enterprises automating trade document processing
* Returns management for international packages

---

## 2. Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/paperlessdocuments/{version}/upload` | Upload trade documents to Forms History |
| `POST` | `/paperlessdocuments/{version}/image` | Push uploaded documents to image repository |
| `DELETE` | `/paperlessdocuments/{version}/DocumentId/ShipperNumber` | Delete specific uploaded document |

### 2.1 Required Headers

| Name | Notes |
|------|-------|
| `transId` | Your own **32-char** correlation ID (UUID/ULID) |
| `transactionSrc` | Free-form identifier for your app (≤ 512 chars). Default `testing` |
| `ShipperNumber` | Your UPS Account Number (6 characters) |

### 2.2 Path Parameters

| Param | Type | Example | Description |
|-------|------|---------|-------------|
| `version` | `'v2'` | `v2` | API version (use v2 for latest features) |

### 2.3 Document Types

| Code | Description |
|------|-------------|
| `001` | Authorization Form |
| `002` | Commercial Invoice |
| `003` | Certificate of Origin |
| `004` | Export Accompanying Document |
| `005` | Export License |
| `006` | Import Permit |
| `007` | One Time NAFTA |
| `008` | Other Document |
| `009` | Power of Attorney |
| `010` | Packing List |
| `011` | SED Document |
| `012` | Shipper's Letter of Instruction |
| `013` | Declaration |

### 2.4 Supported File Formats

`bmp`, `doc`, `docx`, `gif`, `jpg`, `pdf`, `png`, `rtf`, `tif`, `txt`, `xls`, `xlsx`

---

## 3. TypeScript Interface Definitions

```ts
// ---------- OAuth ----------
export interface OAuthToken {
  access_token: string;
  token_type: 'Bearer';
  expires_in: number; // seconds
}

// ---------- Document Upload ----------
export interface TransactionReference {
  CustomerContext?: string;
}

export interface UploadRequest {
  RequestOption?: string;
  SubVersion?: string;
  TransactionReference?: TransactionReference;
}

export interface UserCreatedForm {
  UserCreatedFormFileName: string;          // 1-300 chars
  UserCreatedFormFile: string;              // base64 encoded content
  UserCreatedFormFileFormat: string;        // 3-4 chars (pdf, txt, docx, etc.)
  UserCreatedFormDocumentType: string;      // 3 digits (001-013)
}

export interface UploadRequestBody {
  Request?: UploadRequest;
  ShipperNumber?: string;                   // 6 chars
  UserCreatedForm: UserCreatedForm | UserCreatedForm[];
}

export interface PaperlessUploadRequestWrapper {
  UploadRequest: UploadRequestBody;
}

// ---------- Upload Response ----------
export interface ResponseStatus {
  Code: '1' | '0';                         // 1 = Success, 0 = Failed
  Description: string;                      // "Success" or "Failure"
}

export interface Alert {
  Code: string;                            // 1-10 chars
  Description: string;                     // 1-150 chars
}

export interface UploadResponse {
  ResponseStatus: ResponseStatus;
  Alert?: Alert[];
  TransactionReference?: TransactionReference;
}

export interface FormsHistoryDocumentID {
  DocumentID: string[];                    // Array of 26-char document IDs
}

export interface UploadResponseBody {
  Response: UploadResponse;
  FormsHistoryDocumentID?: FormsHistoryDocumentID;
}

export interface PaperlessUploadResponseWrapper {
  UploadResponse: UploadResponseBody;
}

// ---------- Push to Image Repository ----------
export interface PushToImageRepositoryRequest {
  RequestOption?: string;
  SubVersion?: string;
  TransactionReference?: TransactionReference;
}

export interface PushToImageRepositoryRequestBody {
  Request?: PushToImageRepositoryRequest;
  ShipperNumber?: string;                  // 6 chars
  FormsHistoryDocumentID: {
    DocumentID: string[];                  // Array of 26-char document IDs
  };
  FormsGroupID?: string;                   // 26 chars
  ShipmentIdentifier: string;              // 1-19 chars
  ShipmentDateAndTime?: string;            // yyyy-MM-dd-HH.mm.ss
  ShipmentType: '1' | '2';                // 1 = small package, 2 = freight
  PRQConfirmationNumber?: string;          // 9-30 chars (freight only)
  TrackingNumber?: string[];               // 7-20 chars each (small package only)
}

export interface PaperlessPushRequestWrapper {
  PushToImageRepositoryRequest: PushToImageRepositoryRequestBody;
}

export interface PushToImageRepositoryResponse {
  ResponseStatus: ResponseStatus;
  Alert?: Alert[];
  TransactionReference?: TransactionReference;
}

export interface PushToImageRepositoryResponseBody {
  Response: PushToImageRepositoryResponse;
  FormsGroupID?: string;                   // 26 chars
}

export interface PaperlessPushResponseWrapper {
  PushToImageRepositoryResponse: PushToImageRepositoryResponseBody;
}

// ---------- Delete Document ----------
export interface DeleteRequest {
  RequestOption?: string;
  SubVersion?: string;
  TransactionReference?: TransactionReference;
}

export interface DeleteRequestBody {
  Request?: DeleteRequest;
  ShipperNumber: string;                   // 6 chars
  DocumentID: string;                      // 26 chars
}

export interface PaperlessDeleteRequestWrapper {
  DeleteRequest: DeleteRequestBody;
}

export interface DeleteResponse {
  ResponseStatus: ResponseStatus;
  Alert?: Alert[];
  TransactionReference?: TransactionReference;
}

export interface DeleteResponseBody {
  Response: DeleteResponse;
}

export interface PaperlessDeleteResponseWrapper {
  DeleteResponse: DeleteResponseBody;
}

// ---------- Error Response ----------
export interface ErrorMessage {
  code: string;
  message: string;
}

export interface CommonErrorResponse {
  errors: ErrorMessage[];
}

export interface ErrorResponse {
  response: CommonErrorResponse;
}
```

---

## 4. Production-Ready Node.js Implementation

### 4.1 Install Dependencies

```bash
npm install axios qs
npm install -D @types/node
```

### 4.2 `upsPaperlessClient.ts`

```ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import qs from 'qs';
import * as fs from 'fs';
import {
  OAuthToken,
  PaperlessUploadRequestWrapper,
  PaperlessUploadResponseWrapper,
  PaperlessPushRequestWrapper,
  PaperlessPushResponseWrapper,
  PaperlessDeleteResponseWrapper,
  ErrorResponse,
  UserCreatedForm
} from './types/ups-paperless';

interface UpsPaperlessClientOptions {
  clientId: string;
  clientSecret: string;
  baseURL?: string;               // default CIE
  shipperNumber: string;          // Your UPS Account Number
}

export class UpsPaperlessClient {
  private axios: AxiosInstance;
  private token?: OAuthToken;
  private tokenFetchedAt = 0;

  constructor(private readonly opts: UpsPaperlessClientOptions) {
    this.axios = axios.create({
      baseURL: opts.baseURL ?? 'https://wwwcie.ups.com/api',
      timeout: 30_000 // Longer timeout for file uploads
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
        headers: { 
          'Content-Type': 'application/x-www-form-urlencoded', 
          Authorization: `Basic ${basic}` 
        }
      }
    );
    
    this.token = data;
    this.tokenFetchedAt = now;
    return data.access_token;
  }

  private authHeaders(token: string, transId?: string, transactionSrc?: string) {
    return {
      Authorization: `Bearer ${token}`,
      ShipperNumber: this.opts.shipperNumber,
      transId: transId ?? this.generateTransactionId(),
      transactionSrc: transactionSrc ?? 'paperless-client'
    };
  }

  private generateTransactionId(): string {
    return crypto.randomUUID().replace(/-/g, '');
  }

  // -------- File helper --------
  private fileToBase64(filePath: string): string {
    const fileBuffer = fs.readFileSync(filePath);
    return fileBuffer.toString('base64');
  }

  private getFileExtension(filePath: string): string {
    return filePath.split('.').pop()?.toLowerCase() || '';
  }

  // -------- Public API --------

  /** Upload a document file to UPS Forms History */
  async uploadDocument(
    filePath: string,
    documentType: string,
    options?: {
      customerContext?: string;
      transId?: string;
      transactionSrc?: string;
    }
  ): Promise<PaperlessUploadResponseWrapper> {
    const token = await this.getToken();
    
    const fileName = filePath.split('/').pop() || 'document';
    const fileFormat = this.getFileExtension(filePath);
    const fileContent = this.fileToBase64(filePath);

    const request: PaperlessUploadRequestWrapper = {
      UploadRequest: {
        Request: {
          TransactionReference: {
            CustomerContext: options?.customerContext
          }
        },
        UserCreatedForm: {
          UserCreatedFormFileName: fileName,
          UserCreatedFormFile: fileContent,
          UserCreatedFormFileFormat: fileFormat,
          UserCreatedFormDocumentType: documentType
        }
      }
    };

    const { data } = await this.axios.post<PaperlessUploadResponseWrapper>(
      '/paperlessdocuments/v2/upload',
      request,
      {
        headers: this.authHeaders(token, options?.transId, options?.transactionSrc)
      }
    );

    return data;
  }

  /** Upload multiple documents at once */
  async uploadDocuments(
    documents: Array<{
      filePath: string;
      documentType: string;
    }>,
    options?: {
      customerContext?: string;
      transId?: string;
      transactionSrc?: string;
    }
  ): Promise<PaperlessUploadResponseWrapper> {
    const token = await this.getToken();
    
    const userCreatedForms: UserCreatedForm[] = documents.map(doc => ({
      UserCreatedFormFileName: doc.filePath.split('/').pop() || 'document',
      UserCreatedFormFile: this.fileToBase64(doc.filePath),
      UserCreatedFormFileFormat: this.getFileExtension(doc.filePath),
      UserCreatedFormDocumentType: doc.documentType
    }));

    const request: PaperlessUploadRequestWrapper = {
      UploadRequest: {
        Request: {
          TransactionReference: {
            CustomerContext: options?.customerContext
          }
        },
        UserCreatedForm: userCreatedForms
      }
    };

    const { data } = await this.axios.post<PaperlessUploadResponseWrapper>(
      '/paperlessdocuments/v2/upload',
      request,
      {
        headers: this.authHeaders(token, options?.transId, options?.transactionSrc)
      }
    );

    return data;
  }

  /** Push uploaded documents to image repository for shipment processing */
  async pushToImageRepository(
    documentIds: string[],
    shipmentIdentifier: string,
    shipmentType: '1' | '2',
    options?: {
      formsGroupId?: string;
      shipmentDateAndTime?: string;
      prqConfirmationNumber?: string;
      trackingNumbers?: string[];
      customerContext?: string;
      transId?: string;
      transactionSrc?: string;
    }
  ): Promise<PaperlessPushResponseWrapper> {
    const token = await this.getToken();

    const request: PaperlessPushRequestWrapper = {
      PushToImageRepositoryRequest: {
        Request: {
          TransactionReference: {
            CustomerContext: options?.customerContext
          }
        },
        FormsHistoryDocumentID: {
          DocumentID: documentIds
        },
        FormsGroupID: options?.formsGroupId,
        ShipmentIdentifier: shipmentIdentifier,
        ShipmentDateAndTime: options?.shipmentDateAndTime,
        ShipmentType: shipmentType,
        PRQConfirmationNumber: options?.prqConfirmationNumber,
        TrackingNumber: options?.trackingNumbers
      }
    };

    const { data } = await this.axios.post<PaperlessPushResponseWrapper>(
      '/paperlessdocuments/v2/image',
      request,
      {
        headers: this.authHeaders(token, options?.transId, options?.transactionSrc)
      }
    );

    return data;
  }

  /** Delete a specific document from Forms History */
  async deleteDocument(
    documentId: string,
    options?: {
      transId?: string;
      transactionSrc?: string;
    }
  ): Promise<PaperlessDeleteResponseWrapper> {
    const token = await this.getToken();

    const { data } = await this.axios.delete<PaperlessDeleteResponseWrapper>(
      '/paperlessdocuments/v2/DocumentId/ShipperNumber',
      {
        headers: {
          ...this.authHeaders(token, options?.transId, options?.transactionSrc),
          DocumentId: documentId
        }
      }
    );

    return data;
  }

  /** Get raw axios instance for advanced usage */
  get http() { return this.axios; }
}

// Helper function to parse UPS errors
export function parseUpsError(err: unknown): string {
  if (axios.isAxiosError(err)) {
    const e = err as AxiosError<ErrorResponse>;
    return JSON.stringify(e.response?.data?.response?.errors ?? e.message);
  }
  return 'Unknown error';
}
```

### 4.3 Usage Examples

```ts
import { UpsPaperlessClient, parseUpsError } from './upsPaperlessClient';

// Initialize client
const paperlessClient = new UpsPaperlessClient({
  clientId: process.env.UPS_CLIENT_ID!,
  clientSecret: process.env.UPS_CLIENT_SECRET!,
  shipperNumber: process.env.UPS_SHIPPER_NUMBER!,
  baseURL: process.env.NODE_ENV === 'production'
    ? 'https://onlinetools.ups.com/api'
    : 'https://wwwcie.ups.com/api'
});

// Example 1: Upload a single commercial invoice
async function uploadInvoice() {
  try {
    const response = await paperlessClient.uploadDocument(
      './documents/commercial-invoice.pdf',
      '002', // Commercial Invoice
      {
        customerContext: 'order-12345',
        transactionSrc: 'my-shipping-app'
      }
    );

    const documentIds = response.UploadResponse.FormsHistoryDocumentID?.DocumentID;
    console.log('Document uploaded successfully:', documentIds);
    
    return documentIds;
  } catch (error) {
    console.error('Upload failed:', parseUpsError(error));
    throw error;
  }
}

// Example 2: Upload multiple documents
async function uploadMultipleDocuments() {
  try {
    const response = await paperlessClient.uploadDocuments([
      { filePath: './documents/commercial-invoice.pdf', documentType: '002' },
      { filePath: './documents/certificate-of-origin.pdf', documentType: '003' },
      { filePath: './documents/packing-list.pdf', documentType: '010' }
    ], {
      customerContext: 'shipment-67890'
    });

    const documentIds = response.UploadResponse.FormsHistoryDocumentID?.DocumentID;
    console.log('Documents uploaded successfully:', documentIds);
    
    return documentIds;
  } catch (error) {
    console.error('Multi-upload failed:', parseUpsError(error));
    throw error;
  }
}

// Example 3: Push documents to image repository
async function pushDocumentsToRepository(documentIds: string[]) {
  try {
    const response = await paperlessClient.pushToImageRepository(
      documentIds,
      'SHIP123456789',
      '1', // Small package
      {
        shipmentDateAndTime: '2024-01-15-10.30.45',
        trackingNumbers: ['1Z999AA1234567890'],
        customerContext: 'processing-order-12345'
      }
    );

    console.log('Documents pushed to repository:', response.PushToImageRepositoryResponse.FormsGroupID);
    return response.PushToImageRepositoryResponse.FormsGroupID;
  } catch (error) {
    console.error('Push failed:', parseUpsError(error));
    throw error;
  }
}

// Example 4: Complete workflow
async function completeDocumentWorkflow() {
  try {
    // 1. Upload documents
    const documentIds = await uploadMultipleDocuments();
    
    if (!documentIds || documentIds.length === 0) {
      throw new Error('No documents were uploaded');
    }

    // 2. Push to image repository
    const formsGroupId = await pushDocumentsToRepository(documentIds);

    console.log('Workflow completed successfully');
    console.log('Forms Group ID:', formsGroupId);
    
    return { documentIds, formsGroupId };
  } catch (error) {
    console.error('Workflow failed:', parseUpsError(error));
    throw error;
  }
}

// Example 5: Delete document if needed
async function deleteDocumentExample(documentId: string) {
  try {
    await paperlessClient.deleteDocument(documentId, {
      transactionSrc: 'cleanup-process'
    });
    
    console.log('Document deleted successfully');
  } catch (error) {
    console.error('Delete failed:', parseUpsError(error));
    throw error;
  }
}
```

---

## 5. OAuth2 Authentication Setup

The UPS Paperless Document API uses OAuth 2.0 Client Credentials flow:

```bash
POST https://wwwcie.ups.com/security/v1/oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=client_credentials
```

**Environment URLs:**
- **Test (CIE)**: `https://wwwcie.ups.com`
- **Production**: `https://onlinetools.ups.com`

**Token Management:**
- Tokens are valid for ~1 hour
- Cache and reuse tokens until 60 seconds before expiry
- Implement automatic token refresh in your client

---

## 6. Error Handling Examples

### 6.1 Common Error Scenarios

```ts
import { UpsPaperlessClient, parseUpsError } from './upsPaperlessClient';

async function robustDocumentUpload(filePath: string, documentType: string) {
  const client = new UpsPaperlessClient({
    clientId: process.env.UPS_CLIENT_ID!,
    clientSecret: process.env.UPS_CLIENT_SECRET!,
    shipperNumber: process.env.UPS_SHIPPER_NUMBER!
  });

  try {
    const response = await client.uploadDocument(filePath, documentType);
    return response;
  } catch (error) {
    if (axios.isAxiosError(error)) {
      const status = error.response?.status;
      const upsError = error.response?.data as ErrorResponse;
      
      switch (status) {
        case 400:
          console.error('Validation error:', upsError.response?.errors);
          // Handle specific validation errors
          break;
        case 401:
          console.error('Authentication failed - check credentials');
          break;
        case 403:
          console.error('Access forbidden - check shipper number permissions');
          break;
        case 429:
          console.error('Rate limit exceeded - implement retry with backoff');
          break;
        default:
          console.error('Unexpected error:', parseUpsError(error));
      }
    }
    throw error;
  }
}

// Retry logic for rate limiting
async function uploadWithRetry(filePath: string, documentType: string, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await robustDocumentUpload(filePath, documentType);
    } catch (error) {
      if (axios.isAxiosError(error) && error.response?.status === 429) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        console.log(`Rate limited, retrying in ${delay}ms (attempt ${attempt}/${maxRetries})`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      throw error; // Re-throw non-rate-limit errors
    }
  }
  throw new Error(`Failed after ${maxRetries} attempts`);
}
```

### 6.2 Validation Error Handling

```ts
function validateDocumentType(documentType: string): boolean {
  const validTypes = ['001', '002', '003', '004', '005', '006', '007', '008', '009', '010', '011', '012', '013'];
  return validTypes.includes(documentType);
}

function validateFileFormat(filePath: string): boolean {
  const validFormats = ['bmp', 'doc', 'docx', 'gif', 'jpg', 'pdf', 'png', 'rtf', 'tif', 'txt', 'xls', 'xlsx'];
  const extension = filePath.split('.').pop()?.toLowerCase();
  return validFormats.includes(extension || '');
}

function validateFileSize(filePath: string, maxSizeMB: number = 10): boolean {
  const stats = fs.statSync(filePath);
  const fileSizeMB = stats.size / (1024 * 1024);
  return fileSizeMB <= maxSizeMB;
}

async function validateAndUpload(filePath: string, documentType: string) {
  // Pre-upload validation
  if (!validateDocumentType(documentType)) {
    throw new Error(`Invalid document type: ${documentType}`);
  }
  
  if (!validateFileFormat(filePath)) {
    throw new Error(`Unsupported file format: ${filePath}`);
  }
  
  if (!validateFileSize(filePath)) {
    throw new Error(`File too large: ${filePath}`);
  }
  
  // Upload with validation
  return await uploadWithRetry(filePath, documentType);
}
```

---

## 7. Best Practices for Paperless Document Handling

### 7.1 Document Management

1. **Validate files before upload**
   - Check file size (≤10MB production, ≤1MB test)
   - Verify supported formats
   - Validate document type codes

2. **Organize document types logically**
   ```ts
   export const DOCUMENT_TYPES = {
     AUTHORIZATION_FORM: '001',
     COMMERCIAL_INVOICE: '002',
     CERTIFICATE_OF_ORIGIN: '003',
     EXPORT_LICENSE: '005',
     PACKING_LIST: '010',
     DECLARATION: '013'
   } as const;
   ```

3. **Implement proper error handling**
   - Parse UPS error responses
   - Implement retry logic for transient failures
   - Log correlation IDs for support

### 7.2 Performance Optimization

1. **Batch document uploads**
   ```ts
   // Upload multiple documents in one request
   const documents = [
     { filePath: './invoice.pdf', documentType: '002' },
     { filePath: './origin.pdf', documentType: '003' }
   ];
   await client.uploadDocuments(documents);
   ```

2. **Cache authentication tokens**
   ```ts
   // Token is automatically cached in the client
   const client = new UpsPaperlessClient(config);
   // Multiple calls will reuse the same token
   ```

3. **Optimize file sizes**
   - Compress images appropriately
   - Use PDF for multi-page documents
   - Consider file format based on content type

### 7.3 Security Considerations

1. **Secure credential storage**
   ```ts
   // Use environment variables or secure vaults
   const config = {
     clientId: process.env.UPS_CLIENT_ID!,
     clientSecret: process.env.UPS_CLIENT_SECRET!,
     shipperNumber: process.env.UPS_SHIPPER_NUMBER!
   };
   ```

2. **Audit trail maintenance**
   ```ts
   // Log all document operations
   const transId = generateTransactionId();
   console.log(`Uploading document ${documentType} with transId: ${transId}`);
   ```

3. **Document cleanup**
   ```ts
   // Delete documents when no longer needed
   await client.deleteDocument(documentId);
   ```

### 7.4 Integration Patterns

1. **Workflow automation**
   ```ts
   async function processShipmentDocuments(shipmentId: string) {
     // 1. Upload required documents
     const documentIds = await uploadRequiredDocuments(shipmentId);
     
     // 2. Push to image repository
     const formsGroupId = await pushDocumentsToRepository(documentIds, shipmentId);
     
     // 3. Update shipment with forms group ID
     await updateShipmentWithDocuments(shipmentId, formsGroupId);
   }
   ```

2. **Monitoring and alerting**
   ```ts
   // Monitor upload success rates
   const uploadMetrics = {
     total: 0,
     successful: 0,
     failed: 0
   };
   
   // Alert on high failure rates
   if (uploadMetrics.failed / uploadMetrics.total > 0.1) {
     await sendAlert('High document upload failure rate detected');
   }
   ```

### 7.5 Testing Strategies

1. **Unit tests for validation**
   ```ts
   describe('Document validation', () => {
     it('should validate document types', () => {
       expect(validateDocumentType('002')).toBe(true);
       expect(validateDocumentType('999')).toBe(false);
     });
   });
   ```

2. **Integration tests with test documents**
   ```ts
   // Use small test files in CIE environment
   const testDoc = './test-files/sample-invoice.pdf';
   await client.uploadDocument(testDoc, '002');
   ```

3. **Mock external dependencies**
   ```ts
   // Mock file system operations for testing
   jest.mock('fs');
   const mockFs = fs as jest.Mocked<typeof fs>;
   ```

---

## 8. Migration from v1 to v2

If you're migrating from v1 to v2, note these key differences:

1. **Array responses**: v2 always returns arrays for `DocumentID` and `Alert` fields
2. **Improved error handling**: More detailed error messages and codes
3. **Enhanced validation**: Stricter validation on file formats and sizes

---

Happy shipping with paperless documents! 🚚📋
