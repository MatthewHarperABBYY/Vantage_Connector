# Process Documents - README

## Overview
This folder contains BPMN process diagrams for integrating ABBYY Vantage document processing capabilities with Camunda 8. The processes enable end-to-end intelligent document processing, from file submission through AI-powered extraction to results retrieval.

## Architecture

### Main Process
**ABBYY Vantage E2E** (`ABBYY Vantage E2E.bpmn`)
- Entry point for the complete document processing workflow
- Orchestrates three sub-processes in sequence
- Input: Raw document files (via form)
- Output: Downloaded result files with extracted data

### Sub-Processes

#### 1. Send Files To Vantage (`ABBYY Vantage - Send Files for Processing.bpmn`)
**Purpose:** Upload documents to ABBYY Vantage for processing

**Key Steps:**
- Authenticate with Vantage OAuth2
- Create a new transaction
- Upload files to the transaction (multi-instance loop)
- Start processing

**Input Variables:**
- `files` - Array of file objects to process

**Output Variables:**
- `Transaction.Id` - Vantage transaction identifier
- `Transaction.MobileInputLink` - Optional mobile input link

#### 2. Manage Vantage Processing (`ABBYY Vantage - Get Processing Results.bpmn`)
**Purpose:** Monitor transaction status and handle processing outcomes

**Key Features:**
- Polls Vantage API every 10 seconds (max 30 attempts = 5 minutes)
- Handles three completion states:
  - **Processed** - Successfully completed
  - **Manual Review Required** - Sends link for human review
  - **Failed** - Processing errors or timeout

**Input Variables:**
- `Transaction.Id` - Transaction to monitor

**Output Variables:**
- `TransactionMetadata` - Complete transaction status including:
  - Status
  - Documents array with result file IDs
  - Business rules errors
  - Source files
  - Manual review link (if applicable)

#### 3. Download Vantage Results (`ABBYY Vantage - Download Results.bpmn`)
**Purpose:** Retrieve processed documents and extracted data

**Key Features:**
- Nested multi-instance loops:
  - Outer loop: Iterate through documents
  - Inner loop: Download each result file (JSON, fields JSON, etc.)

**Input Variables:**
- `TransactionData` - Transaction metadata from previous step

**Output Variables:**
- `Results` - Array of downloaded files with metadata:
  - File document ID
  - File name
  - File size
  - Request ID

## Configuration Requirements

### Secrets (Camunda 8 SaaS)
The following secrets must be configured in your Camunda cluster:

- `VANTAGE_BASE_URL` - ABBYY Vantage API endpoint (e.g., `https://vantage-us.abbyy.com`)
- `VANTAGE_CLIENT_ID` - OAuth2 client identifier
- `VANTAGE_CLIENT_SECRET` - OAuth2 client secret

### Skill Configuration
The current implementation uses skill ID: `cb83ff65-3318-44f2-8e13-70ad947542ac`

To use a different skill:
1. Open `ABBYY Vantage - Send Files for Processing.bpmn`
2. Locate the "Create New Transaction" task
3. Update the `skillId` input mapping

## Process Variables Flow

```
Start (User Form)
├─ RawFiles_Vantage (files array)
│
Send Files Process
├─ New_VantageTransactionMetadata
│  ├─ Id
│  └─ mobileInputLink
│
Get Processing Results
├─ Processed_VantageTransactionMetadata
│  ├─ TransactionID
│  ├─ Status
│  ├─ Documents[]
│  │  ├─ id
│  │  ├─ resultFiles[]
│  │  └─ businessRulesErrors[]
│  ├─ SourceFiles[]
│  └─ ManualReviewLink
│
Download Results
└─ ResultFiles_Vantage[]
   ├─ fileDocId
   ├─ fileName
   ├─ size
   └─ requestId
```

## Error Handling

### Authentication Errors
- OAuth token requests retry 3 times with 5-second backoff
- Failed authentication stops the process with error details

### Processing Failures
The "Get Processing Results" process handles:
- **Timeout:** Process ends after 30 polling attempts (5 minutes)
- **Error Status:** Transaction status other than "Processing" or "Processed"
- **Explicit Errors:** Non-null Error field in transaction metadata
- **Manual Review:** Separate end event when ManualReviewLink is present

### Download Errors
- File downloads retry once with no backoff
- Failed downloads terminate the sub-process

## Usage

### Starting the Process
1. Deploy all four BPMN files to your Camunda 8 cluster
2. Start a new instance of "ABBYY Vantage E2E"
3. Upload files via the form
4. Monitor process progress in Operate

### Manual Review Workflow
When documents require human review:
1. Process ends at "Handle Manual Review" event
2. Retrieve `ManualReviewLink` from output variables
3. Send link to reviewers via email (future enhancement)
4. After review, restart or create a new process instance

## Connector Dependencies

### HTTP JSON Connector
All service tasks use the `io.camunda:http-json:1` connector type
- Built-in Camunda 8 SaaS connector
- Supports bearer token authentication
- Handles multipart form data for file uploads

### Custom Element Templates
- `vantage-get-oauth-token` - OAuth2 authentication template
- Additional templates for Vantage-specific operations

## Best Practices

### Performance
- Multi-instance loops execute sequentially to avoid API rate limits
- Connection and read timeouts set to 20 seconds
- Response storage disabled for large file downloads where possible

### Security
- Secrets never exposed in process variables
- OAuth tokens stored in scoped variables (`VantageAuth`)
- Bearer token passed via headers, not query parameters

### Monitoring
- Each major step outputs distinct variables for tracking
- Result expressions provide structured data for downstream processes
- Request IDs included in responses for troubleshooting

## Detailed Variable Structures

### VantageAuth (OAuth Token)
```json
{
  "VantageAuth": {
    "accessToken": "eyJ...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "scope": "openid permissions global.wildcard"
  }
}
```

### Transaction Metadata (Processed)
```json
{
  "TransactionMetadata": {
    "TransactionID": "76800e3c-26f8-4395-853c-8f340b9c6de5",
    "Status": "Processed",
    "ManualReviewLink": null,
    "Error": null,
    "Documents": [
      {
        "id": "28c88be3-9deb-4a32-ac74-179d76e16b5f",
        "resultFiles": [
          {
            "fileId": ".blob.6b4aad1a-11cb-42f9-8b5a-d284d0a95313",
            "fileName": "Invoice.json",
            "type": "Json"
          },
          {
            "fileId": ".blob.7e4543e8-d0b1-458d-92c9-4d81d6e934d7",
            "fileName": "Invoice_fields.json",
            "type": "FieldsJson"
          }
        ],
        "businessRulesErrors": [
          {
            "errorType": "CustomScript",
            "customMessage": "Currency does not match allowed values.",
            "ruleId": "38a17082-20d9-4551-b081-d8467416bb25"
          },
          {
            "errorType": "Normalization"
          }
        ]
      }
    ],
    "SourceFiles": [
      {
        "id": ".blob.affc00cf-e973-4287-bbaf-23869743c695",
        "name": "invoice_1.jpg"
      }
    ]
  }
}
```

### Result Files
```json
{
  "ResultFiles_Vantage": [
    [
      {
        "fileDocId": "28c88be3-9deb-4a32-ac74-179d76e16b5f",
        "fileName": "Invoice.json",
        "size": 15420,
        "requestId": "req-6b4aad1a-11cb-42f9-8b5a-d284d0a95313"
      }
    ]
  ]
}
```

## Deployment Instructions

### Prerequisites
- Camunda 8 SaaS account with active cluster
- ABBYY Vantage account with API access
- BPMN modeler (Camunda Web Modeler or Desktop Modeler)

### Step 1: Configure Secrets
1. Log in to Camunda 8 Console
2. Select your cluster
3. Navigate to **Settings** > **Secrets**
4. Add the three required secrets

### Step 2: Deploy Processes
Deploy in this sequence to avoid missing process definition errors:
1. `ABBYY Vantage - Send Files for Processing.bpmn`
2. `ABBYY Vantage - Get Processing Results.bpmn`
3. `ABBYY Vantage - Download Results.bpmn`
4. `ABBYY Vantage E2E.bpmn`

### Step 3: Test Deployment
1. Navigate to **Operate**
2. Start a new instance of "ABBYY Vantage E2E"
3. Upload a test document
4. Monitor process progression
5. Verify result files are downloaded successfully

## Troubleshooting

### Common Issues

#### OAuth Token Failures
**Symptoms:** Process fails at "Get OAuth Token" task

**Solutions:**
1. Verify `VANTAGE_CLIENT_ID` and `VANTAGE_CLIENT_SECRET` are correct
2. Check OAuth scopes in Vantage console
3. Ensure `VANTAGE_BASE_URL` doesn't include trailing slash

#### Polling Timeout
**Symptoms:** Process ends at "Processing Failed", poll counter reaches 30

**Solutions:**
1. Check document processing time in Vantage console
2. Increase maximum poll attempts if documents take longer
3. Verify transaction was actually submitted to Vantage

#### File Download Errors
**Symptoms:** Process fails at "Download Result File"

**Solutions:**
1. Verify transaction completed successfully
2. Check file IDs are valid in transaction metadata
3. Increase connection/read timeout if files are large

## API Reference

### ABBYY Vantage Endpoints Used

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/auth2/connect/token` | POST | Get OAuth2 token |
| `/api/publicapi/v1/transactions` | POST | Create transaction |
| `/api/publicapi/v1/transactions/{id}/files/` | POST | Upload file |
| `/api/publicapi/v1/transactions/{id}/start` | POST | Start processing |
| `/api/publicapi/v1/transactions/{id}` | GET | Get status |
| `/api/publicapi/v1/transactions/{id}/files/{fileId}/download` | GET | Download result |

## Future Enhancements
- Email notification task for manual review links
- Configurable polling interval and maximum attempts
- Parallel file download support
- Webhook-based status updates instead of polling
- Enhanced business rules error handling

## Support Resources

- [Camunda 8 Documentation](https://docs.camunda.io/)
- [ABBYY Vantage API Docs](https://developers.abbyy.com/)
- [Camunda Forum](https://forum.camunda.io/)

---

**Document Version:** 1.0  
**Last Updated:** 2025  
**Maintainer:** Process Documents Team