# Design Document: BimaSathi

## Overview

BimaSathi is architected as a serverless, event-driven system built entirely on AWS services. The design prioritizes accessibility, scalability, and cost-efficiency while maintaining security and regulatory compliance. The system follows a microservices architecture where each functional capability is implemented as independent Lambda functions orchestrated through Step Functions workflows.

The architecture is organized into five primary layers:

1. **User Access Layer**: WhatsApp Business API, Twilio Voice/IVR, and SMS interfaces
2. **AI Processing Layer**: Amazon Bedrock (LLM), Transcribe (speech-to-text), and Rekognition (image analysis)
3. **Business Logic Layer**: Lambda functions implementing core claim processing workflows
4. **Data Layer**: DynamoDB for structured data, S3 for unstructured data (images, documents)
5. **Integration Layer**: EventBridge for scheduling, SNS for notifications, external API integrations

The system is designed to handle intermittent connectivity, support multiple languages, and provide graceful degradation when services are unavailable. All data is stored in AWS India regions (Mumbai/Hyderabad) for regulatory compliance.

### Key Design Principles

- **Accessibility First**: Voice and conversational interfaces for low-literacy users
- **Serverless Architecture**: Auto-scaling, pay-per-use, minimal operational overhead
- **Event-Driven**: Asynchronous processing for resilience and scalability
- **Security by Design**: Encryption, least-privilege access, audit logging
- **Cost-Optimized**: Efficient resource usage, caching, intelligent storage tiering
- **Resilient**: Graceful degradation, retry logic, fallback mechanisms
- **Observable**: Comprehensive logging, metrics, and distributed tracing


## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER ACCESS LAYER                            │
├─────────────────────────────────────────────────────────────────────┤
│  WhatsApp Business API  │  Twilio Voice/IVR  │  SMS Gateway         │
└────────────┬────────────┴──────────┬──────────┴──────────┬──────────┘
             │                       │                      │
             v                       v                      v
┌─────────────────────────────────────────────────────────────────────┐
│                      API GATEWAY (REST/WebSocket)                    │
└────────────┬────────────────────────────────────────────────────────┘
             │
             v
┌─────────────────────────────────────────────────────────────────────┐
│                      BUSINESS LOGIC LAYER                            │
├─────────────────────────────────────────────────────────────────────┤
│  Lambda Functions:                                                   │
│  - Message Handler      - Claim Processor    - Evidence Validator   │
│  - Voice Handler        - Draft Generator    - Status Tracker       │
│  - Reminder Engine      - Operator API       - Integration Handler  │
└────────────┬────────────────────────────────────────────────────────┘
             │
             v
┌─────────────────────────────────────────────────────────────────────┐
│                         AI PROCESSING LAYER                          │
├─────────────────────────────────────────────────────────────────────┤
│  Amazon Bedrock (LLM)  │  Transcribe  │  Rekognition  │  Translate │
└─────────────────────────────────────────────────────────────────────┘
             │
             v
┌─────────────────────────────────────────────────────────────────────┐
│                           DATA LAYER                                 │
├─────────────────────────────────────────────────────────────────────┤
│  DynamoDB Tables:       │  S3 Buckets:                              │
│  - Users                │  - Evidence Images                         │
│  - Claims               │  - Generated Documents                     │
│  - Sessions             │  - Audit Logs                              │
│  - Policies             │  - Backups                                 │
└─────────────────────────────────────────────────────────────────────┘
             │
             v
┌─────────────────────────────────────────────────────────────────────┐
│                      ORCHESTRATION & EVENTS                          │
├─────────────────────────────────────────────────────────────────────┤
│  Step Functions  │  EventBridge  │  SNS  │  SQS  │  CloudWatch     │
└─────────────────────────────────────────────────────────────────────┘
             │
             v
┌─────────────────────────────────────────────────────────────────────┐
│                      EXTERNAL INTEGRATIONS                           │
├─────────────────────────────────────────────────────────────────────┤
│  Weather APIs  │  Satellite Data  │  Insurance Provider APIs        │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

**Claim Filing Flow (WhatsApp)**:
1. Farmer sends message to WhatsApp Business number
2. WhatsApp webhook triggers API Gateway endpoint
3. API Gateway invokes Message Handler Lambda
4. Message Handler retrieves session context from DynamoDB
5. Message Handler calls Bedrock LLM to understand intent and generate response
6. If image uploaded, Message Handler stores in S3 and triggers Evidence Validator
7. Evidence Validator extracts metadata, calls Rekognition, cross-verifies with Weather API
8. When all data collected, Message Handler triggers Claim Processor Step Function
9. Claim Processor orchestrates Draft Generator, PDF creation, and storage
10. Farmer receives draft for approval via WhatsApp
11. Upon approval, claim status updated and operator notified

**Voice Claim Filing Flow**:
1. Farmer calls IVR number (Twilio)
2. Twilio webhook triggers Voice Handler Lambda
3. Voice Handler streams audio to Amazon Transcribe
4. Transcribe returns text in real-time
5. Voice Handler calls Bedrock LLM to extract structured data
6. Voice Handler synthesizes response using Amazon Polly
7. Extracted data stored in DynamoDB, flow continues as WhatsApp flow

**Deadline Reminder Flow**:
1. EventBridge scheduled rule triggers Reminder Engine Lambda daily
2. Reminder Engine queries DynamoDB for upcoming deadlines
3. For each farmer with approaching deadline, Reminder Engine publishes to SNS topic
4. SNS fans out to WhatsApp, SMS, and Voice channels
5. Delivery status tracked in DynamoDB


## Components and Interfaces

### 1. Message Handler Lambda

**Purpose**: Process incoming WhatsApp messages, maintain conversation context, orchestrate responses

**Inputs**:
- WhatsApp webhook payload (JSON)
  - `from`: Sender phone number
  - `message`: Text content or media reference
  - `timestamp`: Message timestamp
  - `messageId`: Unique message identifier

**Outputs**:
- WhatsApp API response (JSON)
  - `to`: Recipient phone number
  - `message`: Response text
  - `buttons`: Optional interactive buttons
  - `media`: Optional media attachments

**Key Functions**:
- `handleIncomingMessage(payload)`: Main entry point for webhook
- `getOrCreateSession(phoneNumber)`: Retrieve or initialize user session
- `detectIntent(message, context)`: Use LLM to understand user intent
- `routeToHandler(intent, context)`: Dispatch to appropriate sub-handler
- `generateResponse(intent, data, language)`: Create contextual response
- `updateSession(sessionId, newState)`: Persist conversation state

**Dependencies**:
- DynamoDB: Sessions table, Users table
- Bedrock: LLM for intent detection and response generation
- S3: For media downloads
- SNS: For async processing triggers

**Error Handling**:
- Invalid webhook signature: Return 401, log security event
- LLM timeout: Use fallback template responses
- Session corruption: Reset session, notify user
- Rate limit exceeded: Queue message, notify user of delay

### 2. Voice Handler Lambda

**Purpose**: Process IVR calls, convert speech to text, synthesize voice responses

**Inputs**:
- Twilio webhook payload (JSON)
  - `CallSid`: Unique call identifier
  - `From`: Caller phone number
  - `SpeechResult`: Transcribed text (if available)
  - `RecordingUrl`: Audio recording URL

**Outputs**:
- TwiML response (XML)
  - `<Say>`: Text to speak
  - `<Gather>`: Collect user input
  - `<Record>`: Record audio
  - `<Redirect>`: Continue call flow

**Key Functions**:
- `handleCallInitiation(payload)`: Greet caller, detect language
- `processVoiceInput(audioUrl)`: Transcribe using Amazon Transcribe
- `extractStructuredData(text, context)`: Use LLM to parse information
- `synthesizeResponse(text, language)`: Convert text to speech using Polly
- `manageCallFlow(callState)`: Orchestrate multi-turn conversation

**Dependencies**:
- Transcribe: Real-time or batch transcription
- Polly: Text-to-speech synthesis
- Bedrock: LLM for data extraction
- DynamoDB: Call sessions table
- S3: Audio recording storage

**Error Handling**:
- Transcription failure: Ask user to repeat, offer SMS fallback
- Poor audio quality: Provide tips, suggest quieter environment
- Call drop: Save progress, send SMS with resume link
- Language detection failure: Default to Hindi, offer language menu

### 3. Evidence Validator Lambda

**Purpose**: Validate uploaded images, extract metadata, perform AI analysis, cross-verify with external data

**Inputs**:
- S3 event notification (JSON)
  - `bucket`: S3 bucket name
  - `key`: Object key (image path)
  - `claimId`: Associated claim identifier

**Outputs**:
- Validation result (JSON)
  - `isValid`: Boolean validation status
  - `validationScore`: 0-100 confidence score
  - `metadata`: Extracted EXIF data
  - `analysis`: Rekognition results
  - `flags`: List of issues detected
  - `recommendations`: Guidance for farmer

**Key Functions**:
- `extractMetadata(imageUrl)`: Parse EXIF data (GPS, timestamp, device)
- `validateGeolocation(gps, expectedLocation)`: Check location accuracy
- `validateTimestamp(timestamp, claimWindow)`: Verify timing
- `analyzeImage(imageUrl)`: Call Rekognition for content analysis
- `detectCropType(labels)`: Identify crop from image labels
- `detectDamageIndicators(labels)`: Find damage evidence
- `crossVerifyWeather(location, date)`: Check weather data consistency
- `calculateTamperScore(metadata, analysis)`: Detect manipulation
- `generateFeedback(validationResult)`: Create actionable guidance

**Dependencies**:
- Rekognition: Image analysis, label detection
- Weather API: Historical weather data
- DynamoDB: Claims table, Policies table
- S3: Image storage
- SNS: Notification for validation completion

**Error Handling**:
- Missing EXIF data: Flag for manual review, request retake
- Rekognition failure: Retry with exponential backoff, fallback to manual
- Weather API unavailable: Skip cross-verification, log warning
- Invalid image format: Notify user, provide format guidance


### 4. Claim Processor (Step Functions Workflow)

**Purpose**: Orchestrate multi-step claim processing from data collection to submission

**Workflow States**:

1. **ValidateInput**: Check all required fields present
2. **GenerateDraft**: Call Draft Generator Lambda
3. **TranslateDocument**: Translate to farmer's language
4. **GeneratePDF**: Create PDF with embedded evidence
5. **StorePDF**: Upload to S3
6. **UpdateClaimStatus**: Mark as ready for review
7. **NotifyFarmer**: Send draft for approval
8. **WaitForApproval**: Pause until farmer confirms
9. **NotifyOperator**: Alert operator of new submission
10. **UpdateFinalStatus**: Mark as submitted

**State Machine Definition** (simplified):
```json
{
  "StartAt": "ValidateInput",
  "States": {
    "ValidateInput": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ValidateInputFunction",
      "Next": "GenerateDraft",
      "Catch": [{"ErrorEquals": ["ValidationError"], "Next": "NotifyValidationFailure"}]
    },
    "GenerateDraft": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:DraftGeneratorFunction",
      "Next": "TranslateDocument"
    },
    "TranslateDocument": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:TranslateFunction",
      "Next": "GeneratePDF"
    },
    "GeneratePDF": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:PDFGeneratorFunction",
      "Next": "StorePDF"
    },
    "StorePDF": {
      "Type": "Task",
      "Resource": "arn:aws:states:aws-sdk:s3:putObject",
      "Next": "UpdateClaimStatus"
    },
    "UpdateClaimStatus": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:UpdateStatusFunction",
      "Next": "NotifyFarmer"
    },
    "NotifyFarmer": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:NotificationFunction",
      "Next": "WaitForApproval"
    },
    "WaitForApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:waitForTaskToken",
      "TimeoutSeconds": 604800,
      "Next": "NotifyOperator"
    },
    "NotifyOperator": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:OperatorNotificationFunction",
      "Next": "UpdateFinalStatus"
    },
    "UpdateFinalStatus": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:UpdateStatusFunction",
      "End": true
    }
  }
}
```

**Error Handling**:
- Each state has retry configuration (3 attempts, exponential backoff)
- Catch blocks route to error notification states
- Failed workflows trigger operator alerts
- Partial progress saved for resume capability

### 5. Draft Generator Lambda

**Purpose**: Generate structured claim documents from collected data using LLM

**Inputs**:
- Claim data (JSON)
  - `farmerId`: User identifier
  - `policyNumber`: Insurance policy number
  - `cropType`: Type of crop
  - `damageType`: Nature of damage
  - `damageDescription`: Farmer's description
  - `evidenceUrls`: List of image URLs
  - `location`: GPS coordinates
  - `damageDate`: Date of damage occurrence

**Outputs**:
- Claim draft (JSON)
  - `claimNumber`: Generated unique identifier
  - `sections`: Structured document sections
  - `englishVersion`: Official English text
  - `localLanguageVersion`: Translated text
  - `metadata`: Generation timestamp, model version

**Key Functions**:
- `buildPrompt(claimData)`: Create LLM prompt with claim details
- `callLLM(prompt)`: Invoke Bedrock with structured output format
- `validateOutput(draftText)`: Ensure all required fields present
- `formatForInsurer(draft)`: Convert to insurer's required format
- `generateClaimNumber()`: Create unique reference number
- `embedEvidence(draft, imageUrls)`: Link evidence to document sections

**LLM Prompt Template**:
```
You are an expert in Indian crop insurance claims. Generate a formal claim document based on the following information:

Farmer Details:
- Name: {farmerName}
- Policy Number: {policyNumber}
- Location: {village}, {district}, {state}

Crop Information:
- Crop Type: {cropType}
- Area Affected: {areaInAcres} acres
- Sowing Date: {sowingDate}

Damage Details:
- Type of Damage: {damageType}
- Date of Occurrence: {damageDate}
- Description: {damageDescription}

Generate a complete claim document with the following sections:
1. Claim Summary
2. Farmer Declaration
3. Damage Assessment
4. Evidence Description
5. Requested Action

Use formal language appropriate for insurance documentation. Be specific and factual.
```

**Dependencies**:
- Bedrock: LLM for document generation
- DynamoDB: Claims table, Users table, Policies table
- Translate: For multilingual output

**Error Handling**:
- LLM timeout: Retry with shorter prompt, use template fallback
- Invalid output format: Re-prompt with format specification
- Missing required fields: Request additional information from farmer


### 6. Reminder Engine Lambda

**Purpose**: Track deadlines and send automated reminders to farmers

**Trigger**: EventBridge scheduled rule (runs daily at 6 AM IST)

**Processing Logic**:
1. Query DynamoDB for all active policies
2. Calculate days until each deadline (damage reporting, claim filing, document submission)
3. Filter policies with deadlines in 7 days, 3 days, or 1 day
4. For incomplete claims, check last activity timestamp
5. Generate personalized reminder messages
6. Publish to SNS topic for multi-channel delivery
7. Update reminder history in DynamoDB

**Key Functions**:
- `scanPolicies()`: Retrieve all active policies with deadlines
- `calculateDaysUntilDeadline(deadline)`: Compute remaining time
- `shouldSendReminder(policy, reminderHistory)`: Check if reminder needed
- `generateReminderMessage(policy, daysRemaining, language)`: Create personalized text
- `publishReminder(farmerId, message, channels)`: Send via SNS
- `updateReminderHistory(farmerId, reminderType)`: Track sent reminders

**Reminder Message Templates**:
```
7 Days Before (Hindi):
नमस्ते {farmerName}, आपकी फसल बीमा क्लेम की अंतिम तिथि 7 दिन में है। कृपया जल्द से जल्द अपना क्लेम दर्ज करें। मदद के लिए "help" भेजें।

3 Days Before (Marathi):
नमस्कार {farmerName}, तुमच्या पीक विमा दाव्याची अंतिम तारीख 3 दिवसांत आहे। कृपया लवकरात लवकर तुमचा दावा नोंदवा।

1 Day Before (Telugu):
హలో {farmerName}, మీ పంట బీమా క్లెయిమ్ చివరి తేదీ రేపు ఉంది। దయచేసి వెంటనే మీ క్లెయిమ్ దాఖలు చేయండి।
```

**Dependencies**:
- DynamoDB: Policies table, Claims table, ReminderHistory table
- SNS: Multi-channel notification delivery
- EventBridge: Scheduled trigger

**Error Handling**:
- DynamoDB scan timeout: Process in batches, resume from last key
- SNS delivery failure: Retry with exponential backoff, log failure
- Invalid phone number: Mark for manual follow-up

### 7. Operator Dashboard API (Lambda + API Gateway)

**Purpose**: Provide REST API for operator web dashboard

**Endpoints**:

**GET /api/claims**
- Query parameters: `status`, `fromDate`, `toDate`, `district`, `page`, `limit`
- Response: Paginated list of claims with summary information
- Authorization: Cognito JWT token

**GET /api/claims/{claimId}**
- Response: Full claim details including evidence, validation results, history
- Authorization: Operator must have access to claim's jurisdiction

**POST /api/claims/{claimId}/approve**
- Request body: `{ "comments": "string", "approvedAmount": number }`
- Response: Updated claim status
- Side effects: Triggers submission to insurance provider, notifies farmer

**POST /api/claims/{claimId}/reject**
- Request body: `{ "reason": "string", "requiredActions": ["string"] }`
- Response: Updated claim status
- Side effects: Notifies farmer with specific feedback

**POST /api/claims/{claimId}/request-info**
- Request body: `{ "requestedFields": ["string"], "message": "string" }`
- Response: Updated claim status
- Side effects: Sends WhatsApp message to farmer

**GET /api/analytics/dashboard**
- Query parameters: `fromDate`, `toDate`, `district`
- Response: Aggregate statistics and metrics
- Authorization: Operator or admin role

**GET /api/evidence/{claimId}/{imageId}**
- Response: Pre-signed S3 URL for image download
- Authorization: Operator must have access to claim

**Key Functions**:
- `authenticateRequest(token)`: Verify Cognito JWT
- `authorizeAccess(operator, claimId)`: Check jurisdiction permissions
- `queryClaims(filters)`: Retrieve claims from DynamoDB
- `updateClaimStatus(claimId, newStatus, metadata)`: Persist status change
- `triggerInsurerSubmission(claimId)`: Initiate integration workflow
- `generatePresignedUrl(s3Key)`: Create temporary download link
- `calculateAnalytics(filters)`: Aggregate metrics from DynamoDB

**Dependencies**:
- Cognito: Authentication and authorization
- DynamoDB: Claims table, Users table, Operators table
- S3: Evidence image storage
- Step Functions: Trigger insurer submission workflow
- SNS: Farmer notifications

**Error Handling**:
- Invalid token: Return 401 Unauthorized
- Insufficient permissions: Return 403 Forbidden
- Claim not found: Return 404 Not Found
- Concurrent modification: Return 409 Conflict with retry guidance
- Server error: Return 500, log error, alert on-call


### 8. Integration Handler Lambda

**Purpose**: Submit approved claims to insurance provider systems

**Inputs**:
- Claim submission event (JSON)
  - `claimId`: Claim identifier
  - `insurerCode`: Insurance provider identifier
  - `submissionType`: "API" or "EMAIL"

**Outputs**:
- Submission result (JSON)
  - `success`: Boolean
  - `insurerReference`: Provider's acknowledgment number
  - `submissionTimestamp`: ISO 8601 timestamp
  - `errorDetails`: If failed

**Key Functions**:
- `getInsurerConfig(insurerCode)`: Retrieve API endpoint, credentials, format
- `formatClaimData(claim, insurerFormat)`: Transform to provider's schema
- `submitViaAPI(claimData, config)`: POST to insurer's REST API
- `submitViaEmail(claimData, config)`: Send email with PDF attachment
- `handleResponse(response)`: Parse acknowledgment, extract reference
- `retryWithBackoff(operation, maxAttempts)`: Exponential backoff retry logic
- `updateClaimWithInsurerRef(claimId, reference)`: Store acknowledgment

**Insurer API Integration Example**:
```javascript
async function submitViaAPI(claimData, config) {
  const payload = {
    policyNumber: claimData.policyNumber,
    claimDate: claimData.submissionDate,
    farmerDetails: {
      name: claimData.farmerName,
      mobile: claimData.farmerMobile,
      aadhaar: claimData.aadhaarHash // Hashed, not plain
    },
    cropDetails: {
      cropType: claimData.cropType,
      areaAffected: claimData.areaInAcres,
      damageType: claimData.damageType
    },
    evidenceUrls: claimData.evidenceUrls.map(url => generatePresignedUrl(url)),
    documentUrl: generatePresignedUrl(claimData.pdfUrl)
  };

  const response = await axios.post(config.apiEndpoint, payload, {
    headers: {
      'Authorization': `Bearer ${config.apiKey}`,
      'Content-Type': 'application/json'
    },
    timeout: 30000
  });

  return {
    success: true,
    insurerReference: response.data.claimReferenceNumber,
    submissionTimestamp: new Date().toISOString()
  };
}
```

**Email Submission Format**:
- To: Insurer's claim submission email
- Subject: `CROP_CLAIM_${claimNumber}_${policyNumber}`
- Body: Plain text summary with claim details
- Attachments: PDF claim document, evidence photos (zipped)

**Dependencies**:
- DynamoDB: Claims table, InsurerConfigs table
- S3: Claim documents, evidence images
- SES: Email sending
- Secrets Manager: API credentials

**Error Handling**:
- API timeout: Retry 3 times with exponential backoff
- Authentication failure: Alert admin, mark for manual submission
- Invalid data format: Log error, notify operator for correction
- Network error: Queue for retry, alert if persistent
- Email bounce: Retry with alternative email, escalate to operator

### 9. Status Tracker Lambda

**Purpose**: Poll insurance provider APIs for claim status updates

**Trigger**: EventBridge scheduled rule (runs daily)

**Processing Logic**:
1. Query DynamoDB for claims in "Sent_to_Insurer" status
2. For each claim, retrieve insurer reference and config
3. Call insurer's status API with reference number
4. Parse response and map to internal status
5. If status changed, update DynamoDB and notify farmer
6. If payout processed, extract amount and date

**Key Functions**:
- `getClaimsAwaitingUpdate()`: Query claims needing status check
- `pollInsurerStatus(insurerRef, config)`: Call status API
- `mapInsurerStatus(externalStatus)`: Convert to internal status enum
- `detectStatusChange(currentStatus, newStatus)`: Identify updates
- `updateClaimStatus(claimId, newStatus, metadata)`: Persist change
- `notifyFarmer(farmerId, statusUpdate)`: Send WhatsApp/SMS notification

**Status Mapping**:
```javascript
const statusMap = {
  'SUBMITTED': 'Under_Review',
  'UNDER_REVIEW': 'Under_Review',
  'APPROVED': 'Approved',
  'REJECTED': 'Rejected',
  'PAYOUT_INITIATED': 'Payout_Processed',
  'PAYOUT_COMPLETED': 'Payout_Processed',
  'ADDITIONAL_INFO_REQUIRED': 'Pending_Information'
};
```

**Dependencies**:
- DynamoDB: Claims table, InsurerConfigs table
- SNS: Farmer notifications
- Secrets Manager: API credentials

**Error Handling**:
- API unavailable: Skip this cycle, retry next day
- Invalid reference: Mark claim for manual investigation
- Parsing error: Log raw response, alert admin


## Data Models

### DynamoDB Table Schemas

#### Users Table

**Table Name**: `bimasathi-users-{environment}`
**Primary Key**: `userId` (String) - Phone number in E.164 format (+91XXXXXXXXXX)
**GSI**: `aadhaarHash-index` on `aadhaarHash` (if Aadhaar used)

**Attributes**:
```json
{
  "userId": "+919876543210",
  "name": "Ramesh Kumar",
  "language": "hi",
  "district": "Pune",
  "state": "Maharashtra",
  "registrationDate": "2024-01-15T10:30:00Z",
  "aadhaarHash": "sha256_hash_value",
  "consentGiven": true,
  "consentTimestamp": "2024-01-15T10:30:00Z",
  "consentVersion": "v1.0",
  "lastActiveDate": "2024-03-20T14:22:00Z",
  "preferences": {
    "notificationChannels": ["whatsapp", "sms"],
    "reminderFrequency": "standard"
  },
  "metadata": {
    "deviceType": "feature_phone",
    "appVersion": "1.0.0"
  }
}
```

#### Claims Table

**Table Name**: `bimasathi-claims-{environment}`
**Primary Key**: `claimId` (String) - UUID
**GSI 1**: `userId-submissionDate-index` on `userId` (Hash) and `submissionDate` (Range)
**GSI 2**: `status-submissionDate-index` on `status` (Hash) and `submissionDate` (Range)
**GSI 3**: `policyNumber-index` on `policyNumber`

**Attributes**:
```json
{
  "claimId": "CLM-2024-001234",
  "userId": "+919876543210",
  "policyNumber": "PMFBY-MH-2024-567890",
  "status": "Under_Review",
  "submissionDate": "2024-03-20T14:22:00Z",
  "lastUpdatedDate": "2024-03-21T09:15:00Z",
  "farmerDetails": {
    "name": "Ramesh Kumar",
    "village": "Shirur",
    "district": "Pune",
    "state": "Maharashtra",
    "mobile": "+919876543210"
  },
  "cropDetails": {
    "cropType": "Cotton",
    "areaInAcres": 2.5,
    "sowingDate": "2024-06-15",
    "expectedHarvestDate": "2024-11-30"
  },
  "damageDetails": {
    "damageType": "Flood",
    "damageDate": "2024-09-10",
    "damageDescription": "Heavy rainfall caused waterlogging, crop submerged for 3 days",
    "estimatedLossPercentage": 75
  },
  "evidence": [
    {
      "imageId": "IMG-001",
      "s3Key": "evidence/CLM-2024-001234/IMG-001.jpg",
      "uploadTimestamp": "2024-03-20T14:25:00Z",
      "metadata": {
        "gpsCoordinates": {"lat": 18.6298, "lon": 73.9925},
        "captureTimestamp": "2024-09-10T16:30:00Z",
        "deviceModel": "Samsung Galaxy M12"
      },
      "validationResult": {
        "isValid": true,
        "validationScore": 92,
        "flags": [],
        "rekognitionLabels": ["Crop", "Water", "Field", "Flood"]
      }
    }
  ],
  "validationSummary": {
    "overallScore": 88,
    "locationVerified": true,
    "timestampVerified": true,
    "weatherCrossVerified": true,
    "tamperScore": 5
  },
  "draftDocument": {
    "s3Key": "drafts/CLM-2024-001234/draft.pdf",
    "generatedDate": "2024-03-20T14:30:00Z",
    "englishVersion": "s3://bucket/drafts/CLM-2024-001234/draft_en.pdf",
    "localLanguageVersion": "s3://bucket/drafts/CLM-2024-001234/draft_hi.pdf"
  },
  "approvalDetails": {
    "farmerApprovedDate": "2024-03-20T15:00:00Z",
    "operatorId": "OP-12345",
    "operatorApprovedDate": "2024-03-21T09:15:00Z",
    "operatorComments": "All documents verified, evidence clear"
  },
  "insurerSubmission": {
    "submittedDate": "2024-03-21T09:20:00Z",
    "insurerCode": "PMFBY-MH",
    "insurerReference": "INS-REF-987654",
    "submissionMethod": "API"
  },
  "payoutDetails": {
    "approvedAmount": 37500,
    "payoutDate": "2024-04-05T00:00:00Z",
    "paymentReference": "PAY-123456"
  },
  "deadlines": {
    "damageReportingDeadline": "2024-09-17T23:59:59Z",
    "claimFilingDeadline": "2024-09-24T23:59:59Z",
    "documentSubmissionDeadline": "2024-10-01T23:59:59Z"
  },
  "history": [
    {
      "timestamp": "2024-03-20T14:22:00Z",
      "status": "Draft",
      "actor": "Farmer",
      "action": "Claim initiated"
    },
    {
      "timestamp": "2024-03-20T15:00:00Z",
      "status": "Submitted",
      "actor": "Farmer",
      "action": "Claim submitted for review"
    },
    {
      "timestamp": "2024-03-21T09:15:00Z",
      "status": "Approved",
      "actor": "Operator-OP-12345",
      "action": "Claim approved and sent to insurer"
    }
  ]
}
```


#### Sessions Table

**Table Name**: `bimasathi-sessions-{environment}`
**Primary Key**: `sessionId` (String) - UUID
**GSI**: `userId-lastActivityTime-index` on `userId` (Hash) and `lastActivityTime` (Range)
**TTL**: `expirationTime` (Number) - Unix timestamp

**Attributes**:
```json
{
  "sessionId": "SESS-uuid-12345",
  "userId": "+919876543210",
  "channel": "whatsapp",
  "language": "hi",
  "currentFlow": "claim_filing",
  "currentStep": "evidence_collection",
  "collectedData": {
    "cropType": "Cotton",
    "damageType": "Flood",
    "damageDate": "2024-09-10",
    "damageDescription": "Heavy rainfall caused waterlogging",
    "evidenceCount": 2
  },
  "context": {
    "lastQuestion": "Please upload photos of the damaged crop",
    "expectedInput": "image",
    "retryCount": 0
  },
  "createdTime": "2024-03-20T14:20:00Z",
  "lastActivityTime": "2024-03-20T14:28:00Z",
  "expirationTime": 1711036080
}
```

#### Policies Table

**Table Name**: `bimasathi-policies-{environment}`
**Primary Key**: `policyNumber` (String)
**GSI**: `userId-index` on `userId`

**Attributes**:
```json
{
  "policyNumber": "PMFBY-MH-2024-567890",
  "userId": "+919876543210",
  "insurerCode": "PMFBY-MH",
  "cropType": "Cotton",
  "areaInsured": 2.5,
  "sumInsured": 50000,
  "premium": 750,
  "season": "Kharif-2024",
  "startDate": "2024-06-01T00:00:00Z",
  "endDate": "2024-12-31T23:59:59Z",
  "landDetails": {
    "surveyNumber": "123/4A",
    "village": "Shirur",
    "district": "Pune",
    "state": "Maharashtra",
    "gpsCoordinates": {"lat": 18.6298, "lon": 73.9925}
  },
  "deadlines": {
    "damageReportingWindow": 72,
    "claimFilingWindow": 7,
    "documentSubmissionWindow": 14
  },
  "status": "Active",
  "claimsHistory": ["CLM-2024-001234"]
}
```

#### ReminderHistory Table

**Table Name**: `bimasathi-reminders-{environment}`
**Primary Key**: `userId` (String) and `reminderDate` (String) - Composite key
**TTL**: `expirationTime` (Number) - 90 days retention

**Attributes**:
```json
{
  "userId": "+919876543210",
  "reminderDate": "2024-03-15",
  "reminderType": "deadline_7days",
  "policyNumber": "PMFBY-MH-2024-567890",
  "sentTimestamp": "2024-03-15T06:00:00Z",
  "channels": ["whatsapp", "sms"],
  "deliveryStatus": {
    "whatsapp": "delivered",
    "sms": "delivered"
  },
  "expirationTime": 1719388800
}
```

#### Operators Table

**Table Name**: `bimasathi-operators-{environment}`
**Primary Key**: `operatorId` (String)
**GSI**: `email-index` on `email`

**Attributes**:
```json
{
  "operatorId": "OP-12345",
  "name": "Suresh Patil",
  "email": "suresh.patil@csc.gov.in",
  "mobile": "+919123456789",
  "role": "CSC_OPERATOR",
  "jurisdiction": {
    "state": "Maharashtra",
    "districts": ["Pune", "Satara"]
  },
  "cognitoUsername": "suresh.patil",
  "status": "Active",
  "registrationDate": "2024-01-10T00:00:00Z",
  "lastLoginDate": "2024-03-21T08:30:00Z",
  "statistics": {
    "totalClaimsReviewed": 156,
    "averageReviewTimeMinutes": 4.5,
    "approvalRate": 0.87
  }
}
```

### S3 Bucket Structure

**Bucket Name**: `bimasathi-storage-{environment}-{region}`

**Directory Structure**:
```
/evidence/
  /{claimId}/
    /IMG-001.jpg
    /IMG-002.jpg
    /metadata.json

/drafts/
  /{claimId}/
    /draft_en.pdf
    /draft_hi.pdf
    /draft_mr.pdf

/final-documents/
  /{claimId}/
    /claim-document.pdf
    /evidence-package.zip

/audit-logs/
  /year=2024/
    /month=03/
      /day=20/
        /logs-{timestamp}.json

/backups/
  /dynamodb/
    /users/
      /backup-{timestamp}/
    /claims/
      /backup-{timestamp}/

/temp/
  /uploads/
    /{sessionId}/
      /temp-image.jpg
```

**Lifecycle Policies**:
- Evidence images: Transition to S3 Glacier after 90 days
- Draft documents: Delete after 30 days if claim not submitted
- Final documents: Retain for 7 years (regulatory requirement)
- Audit logs: Transition to Glacier after 90 days, retain for 7 years
- Temp uploads: Delete after 24 hours
- Backups: Retain for 35 days

**Encryption**:
- All objects encrypted at rest using AWS KMS
- Separate KMS keys for evidence, documents, and audit logs
- Key rotation enabled annually


## LLM Prompt Design Strategy

### Prompt Engineering Principles

1. **Structured Output**: Always request JSON format for programmatic parsing
2. **Few-Shot Learning**: Include 2-3 examples in prompts for consistency
3. **Language Awareness**: Specify input and output language explicitly
4. **Error Handling**: Instruct model on handling ambiguous or incomplete input
5. **Token Optimization**: Use concise prompts to minimize costs
6. **Context Management**: Include only relevant conversation history

### Intent Detection Prompt

**Purpose**: Classify user message into actionable intents

**Prompt Template**:
```
You are BimaSathi, an AI assistant for crop insurance claims. Analyze the user's message and classify their intent.

User Language: {language}
User Message: "{userMessage}"
Conversation Context: {recentHistory}

Classify the intent as one of:
- new_claim: User wants to file a new claim
- check_status: User wants to know claim status
- upload_evidence: User is providing photos or documents
- help: User needs assistance or has questions
- confirm: User is confirming or approving something
- cancel: User wants to cancel or go back
- other: None of the above

Respond in JSON format:
{
  "intent": "intent_name",
  "confidence": 0.95,
  "entities": {
    "claimId": "if mentioned",
    "cropType": "if mentioned"
  },
  "suggestedResponse": "brief response in {language}"
}
```

**Example Input/Output**:
```
Input: "मेरी कपास की फसल बाढ़ में बर्बाद हो गई" (Hindi: My cotton crop was destroyed in flood)

Output:
{
  "intent": "new_claim",
  "confidence": 0.98,
  "entities": {
    "cropType": "Cotton",
    "damageType": "Flood"
  },
  "suggestedResponse": "मुझे आपकी मदद करने में खुशी होगी। कृपया बताएं कि नुकसान कब हुआ?"
}
```

### Data Extraction Prompt

**Purpose**: Extract structured information from conversational input

**Prompt Template**:
```
Extract crop insurance claim information from the user's description.

User Language: {language}
User Input: "{userInput}"

Extract the following fields:
- cropType: Type of crop (Rice, Wheat, Cotton, etc.)
- damageType: Nature of damage (Flood, Drought, Pest, Hail, etc.)
- damageDate: When damage occurred (date or relative time)
- areaAffected: Approximate area in acres
- damageDescription: Brief description of what happened

If a field is not mentioned, set it to null.
If information is ambiguous, include "clarificationNeeded" field.

Respond in JSON format:
{
  "cropType": "value or null",
  "damageType": "value or null",
  "damageDate": "YYYY-MM-DD or null",
  "areaAffected": number or null,
  "damageDescription": "text or null",
  "clarificationNeeded": ["list of fields needing clarification"],
  "confidence": 0.85
}
```

**Example**:
```
Input: "पिछले हफ्ते भारी बारिश में मेरे 2 एकड़ कपास के खेत में पानी भर गया" 
(Last week heavy rain flooded my 2 acre cotton field)

Output:
{
  "cropType": "Cotton",
  "damageType": "Flood",
  "damageDate": "2024-03-13",
  "areaAffected": 2,
  "damageDescription": "Heavy rainfall caused waterlogging in cotton field",
  "clarificationNeeded": [],
  "confidence": 0.92
}
```

### Claim Document Generation Prompt

**Purpose**: Generate formal claim document from structured data

**Prompt Template**:
```
Generate a formal crop insurance claim document in {language}.

Claim Information:
- Farmer Name: {farmerName}
- Policy Number: {policyNumber}
- Crop Type: {cropType}
- Area Affected: {areaInAcres} acres
- Damage Type: {damageType}
- Damage Date: {damageDate}
- Description: {damageDescription}
- Location: {village}, {district}, {state}

Generate a complete claim document with these sections:
1. Claim Reference and Date
2. Farmer Details
3. Policy Information
4. Crop and Land Details
5. Damage Description
6. Evidence Summary
7. Farmer Declaration
8. Requested Action

Use formal, professional language appropriate for insurance documentation.
Be specific and factual. Do not add information not provided.

Respond in JSON format:
{
  "sections": {
    "claimReference": "text",
    "farmerDetails": "text",
    "policyInformation": "text",
    "cropDetails": "text",
    "damageDescription": "text",
    "evidenceSummary": "text",
    "declaration": "text",
    "requestedAction": "text"
  },
  "fullDocument": "complete formatted text"
}
```

### Conversational Response Generation Prompt

**Purpose**: Generate natural, helpful responses in user's language

**Prompt Template**:
```
You are BimaSathi, a friendly AI assistant helping Indian farmers with crop insurance claims.

User Language: {language}
Current Step: {currentStep}
User Message: "{userMessage}"
System State: {systemState}

Generate a helpful response that:
1. Acknowledges the user's input
2. Provides clear next steps
3. Uses simple, conversational language
4. Shows empathy and support
5. Keeps response under 160 characters if possible (for SMS compatibility)

Respond in JSON format:
{
  "message": "response text in {language}",
  "suggestedActions": ["button1", "button2"],
  "requiresInput": true/false
}
```

**Example**:
```
Input:
- Language: Hindi
- Current Step: evidence_collection
- User Message: "फोटो कैसे लेनी है?" (How to take photo?)

Output:
{
  "message": "फोटो लेते समय ध्यान रखें: 1) खेत का पूरा नज़ारा दिखे 2) नुकसान साफ दिखे 3) कोई पहचान चिन्ह (पेड़, मकान) दिखे। क्या आप अभी फोटो लेना चाहेंगे?",
  "suggestedActions": ["हां, फोटो भेजें", "और मदद चाहिए"],
  "requiresInput": true
}
```

### Token Optimization Strategies

1. **Prompt Caching**: Cache system instructions and examples
2. **Context Pruning**: Include only last 3-5 conversation turns
3. **Abbreviations**: Use short field names in JSON
4. **Batch Processing**: Combine multiple extractions in single call
5. **Model Selection**: Use smaller models for simple tasks (intent detection), larger for complex (document generation)


## Evidence Intelligence Validation Logic

### Multi-Layer Validation Pipeline

Evidence validation occurs in four sequential layers, each adding confidence to the overall assessment:

**Layer 1: Metadata Validation**
- Extract EXIF data from uploaded images
- Verify presence of GPS coordinates, timestamp, device information
- Check for editing software signatures in metadata
- Validate timestamp is within reasonable claim window

**Layer 2: Geospatial Validation**
- Compare GPS coordinates with farmer's registered land location
- Calculate distance between photo location and policy location
- Flag if distance exceeds 500 meters threshold
- Check for GPS coordinate consistency across multiple photos
- Detect potential GPS spoofing (impossible movement patterns)

**Layer 3: Visual Content Analysis**
- Use Amazon Rekognition to detect labels and objects
- Identify crop type from visual features
- Detect damage indicators (water, wilting, discoloration, pests)
- Assess image quality (blur, lighting, resolution)
- Detect presence of landmarks or reference objects

**Layer 4: Cross-Verification**
- Query Weather API for historical data at location and date
- Compare claimed damage type with weather events
- Check satellite imagery if available
- Validate consistency across multiple evidence pieces

### Validation Scoring Algorithm

```javascript
function calculateValidationScore(evidence) {
  let score = 100;
  const flags = [];
  
  // Metadata checks (30 points)
  if (!evidence.metadata.gps) {
    score -= 15;
    flags.push('missing_gps');
  }
  if (!evidence.metadata.timestamp) {
    score -= 10;
    flags.push('missing_timestamp');
  }
  if (evidence.metadata.editingSoftware) {
    score -= 5;
    flags.push('edited_image');
  }
  
  // Geospatial checks (25 points)
  const distance = calculateDistance(
    evidence.metadata.gps,
    policy.landLocation
  );
  if (distance > 500) {
    score -= 20;
    flags.push('location_mismatch');
  } else if (distance > 200) {
    score -= 10;
    flags.push('location_uncertain');
  }
  
  // Timestamp checks (20 points)
  const timeDiff = Math.abs(
    evidence.metadata.timestamp - evidence.uploadTimestamp
  );
  if (timeDiff > 30 * 24 * 60 * 60 * 1000) { // 30 days
    score -= 15;
    flags.push('old_photo');
  }
  if (evidence.metadata.timestamp > claim.damageDate + 7 * 24 * 60 * 60 * 1000) {
    score -= 10;
    flags.push('photo_after_damage_date');
  }
  
  // Visual content checks (15 points)
  const cropDetected = evidence.rekognitionLabels.some(
    label => label.Name.toLowerCase().includes(claim.cropType.toLowerCase())
  );
  if (!cropDetected) {
    score -= 10;
    flags.push('crop_not_visible');
  }
  
  const damageIndicators = ['water', 'flood', 'dry', 'pest', 'damage'];
  const damageDetected = evidence.rekognitionLabels.some(
    label => damageIndicators.some(indicator => 
      label.Name.toLowerCase().includes(indicator)
    )
  );
  if (!damageDetected) {
    score -= 5;
    flags.push('damage_not_clear');
  }
  
  // Weather cross-verification (10 points)
  if (weatherData && !weatherData.matchesDamageType) {
    score -= 10;
    flags.push('weather_mismatch');
  }
  
  return {
    score: Math.max(0, score),
    flags: flags,
    recommendation: score >= 70 ? 'approve' : score >= 50 ? 'review' : 'reject'
  };
}
```

### Tamper Detection

```javascript
function detectTampering(evidence) {
  let tamperScore = 0;
  const indicators = [];
  
  // Check for metadata inconsistencies
  if (evidence.metadata.editingSoftware) {
    tamperScore += 20;
    indicators.push('editing_software_detected');
  }
  
  // Check for GPS anomalies
  if (evidence.metadata.gps && isGPSImprobable(evidence.metadata.gps)) {
    tamperScore += 30;
    indicators.push('gps_spoofing_suspected');
  }
  
  // Check timestamp manipulation
  const createDate = evidence.metadata.createDate;
  const modifyDate = evidence.metadata.modifyDate;
  if (modifyDate && modifyDate > createDate + 60000) { // 1 minute
    tamperScore += 15;
    indicators.push('timestamp_modified');
  }
  
  // Check for image manipulation artifacts using Rekognition
  if (evidence.rekognitionModeration) {
    const manipulationConfidence = evidence.rekognitionModeration
      .find(m => m.Name === 'Manipulated')?.Confidence || 0;
    if (manipulationConfidence > 50) {
      tamperScore += manipulationConfidence / 2;
      indicators.push('visual_manipulation_detected');
    }
  }
  
  // Check for duplicate images (same hash)
  if (isDuplicateImage(evidence.imageHash)) {
    tamperScore += 25;
    indicators.push('duplicate_image');
  }
  
  return {
    tamperScore: Math.min(100, tamperScore),
    indicators: indicators,
    isSuspicious: tamperScore > 40
  };
}
```

### Weather Cross-Verification

```javascript
async function crossVerifyWeather(claim, evidence) {
  const weatherData = await weatherAPI.getHistoricalData({
    latitude: evidence.metadata.gps.lat,
    longitude: evidence.metadata.gps.lon,
    date: claim.damageDate,
    daysRange: 3 // Check 3 days around damage date
  });
  
  const damageTypeWeatherMap = {
    'Flood': ['heavy_rain', 'extreme_rain', 'storm'],
    'Drought': ['no_rain', 'low_rainfall', 'high_temperature'],
    'Hail': ['hailstorm', 'severe_weather'],
    'Wind': ['high_wind', 'cyclone', 'storm'],
    'Pest': null // Cannot verify via weather
  };
  
  const expectedWeather = damageTypeWeatherMap[claim.damageType];
  
  if (!expectedWeather) {
    return { verified: null, confidence: 0 };
  }
  
  const weatherMatch = weatherData.events.some(event =>
    expectedWeather.includes(event.type)
  );
  
  const rainfallMatch = claim.damageType === 'Flood' && 
    weatherData.rainfall > 100; // mm
  
  return {
    verified: weatherMatch || rainfallMatch,
    confidence: weatherMatch ? 0.9 : rainfallMatch ? 0.7 : 0.3,
    weatherEvents: weatherData.events,
    rainfall: weatherData.rainfall,
    temperature: weatherData.temperature
  };
}
```

### Feedback Generation

```javascript
function generateEvidenceFeedback(validationResult, language) {
  const feedbackTemplates = {
    'missing_gps': {
      'hi': 'फोटो में GPS जानकारी नहीं है। कृपया अपने फोन में Location सेवा चालू करें और फिर से फोटो लें।',
      'en': 'Photo is missing GPS information. Please enable Location services on your phone and retake the photo.'
    },
    'location_mismatch': {
      'hi': 'फोटो का स्थान आपके खेत से मेल नहीं खाता। कृपया सही खेत की फोटो भेजें।',
      'en': 'Photo location does not match your registered field. Please send photos from the correct field.'
    },
    'crop_not_visible': {
      'hi': 'फोटो में फसल साफ नहीं दिख रही। कृपया नज़दीक से और अच्छी रोशनी में फोटो लें।',
      'en': 'Crop is not clearly visible in photo. Please take a closer photo in good lighting.'
    },
    'damage_not_clear': {
      'hi': 'नुकसान साफ नहीं दिख रहा। कृपया क्षतिग्रस्त हिस्से की स्पष्ट फोटो भेजें।',
      'en': 'Damage is not clearly visible. Please send clear photos of the damaged areas.'
    }
  };
  
  const messages = validationResult.flags
    .map(flag => feedbackTemplates[flag]?.[language])
    .filter(msg => msg);
  
  if (messages.length === 0 && validationResult.score >= 70) {
    return language === 'hi' 
      ? 'बहुत अच्छा! आपकी फोटो सही है।'
      : 'Great! Your photo is valid.';
  }
  
  return messages.join(' ');
}
```


## Security Architecture

### Authentication and Authorization

**User Authentication**:
- Primary: OTP-based verification via SMS
- Phone number as unique identifier (E.164 format)
- OTP valid for 10 minutes, 6-digit numeric code
- Rate limiting: Max 3 OTP requests per hour per number
- Session tokens stored in DynamoDB with 24-hour TTL
- Optional Aadhaar authentication for enhanced verification

**Operator Authentication**:
- AWS Cognito User Pools for operator accounts
- Email + password with MFA (TOTP or SMS)
- JWT tokens with 8-hour expiration
- Refresh tokens with 30-day expiration
- Role-based access control (RBAC)

**Authorization Model**:
```javascript
const roles = {
  FARMER: {
    permissions: [
      'claim:create',
      'claim:read:own',
      'claim:update:own',
      'evidence:upload:own'
    ]
  },
  HELPER: {
    permissions: [
      'claim:create:assisted',
      'claim:read:assisted',
      'evidence:upload:assisted'
    ],
    requiresConsent: true
  },
  CSC_OPERATOR: {
    permissions: [
      'claim:read:jurisdiction',
      'claim:approve:jurisdiction',
      'claim:reject:jurisdiction',
      'analytics:view:jurisdiction'
    ],
    jurisdiction: ['district', 'state']
  },
  ADMIN: {
    permissions: ['*'],
    requiresMFA: true
  }
};
```

### Data Encryption

**At Rest**:
- All DynamoDB tables encrypted using AWS KMS
- Separate KMS keys for different data classifications:
  - `bimasathi-pii-key`: Personal identifiable information
  - `bimasathi-evidence-key`: Photos and documents
  - `bimasathi-audit-key`: Audit logs
- S3 bucket encryption using SSE-KMS
- Automatic key rotation enabled (annual)

**In Transit**:
- TLS 1.2 minimum for all API communications
- Certificate pinning for mobile clients
- VPC endpoints for AWS service communication
- No data transmitted over unencrypted channels

**Sensitive Data Handling**:
```javascript
// Aadhaar hashing (never store plain text)
function hashAadhaar(aadhaarNumber) {
  return crypto
    .createHash('sha256')
    .update(aadhaarNumber + process.env.AADHAAR_SALT)
    .digest('hex');
}

// PII masking in logs
function maskPII(data) {
  return {
    ...data,
    mobile: data.mobile?.replace(/(\d{2})\d{8}(\d{2})/, '$1********$2'),
    aadhaar: data.aadhaar ? '****-****-' + data.aadhaar.slice(-4) : null,
    name: data.name?.replace(/(?<=.).(?=.)/g, '*')
  };
}
```

### IAM Strategy

**Principle of Least Privilege**:
- Each Lambda function has dedicated IAM role
- Roles grant only required permissions
- No wildcard permissions in production
- Resource-level permissions where possible

**Example Lambda IAM Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:ap-south-1:*:table/bimasathi-claims-prod"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::bimasathi-storage-prod/evidence/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:ap-south-1:*:key/evidence-key-id"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:ap-south-1:*:log-group:/aws/lambda/bimasathi-*"
    }
  ]
}
```

**Service Control Policies**:
- Enforce data localization (India regions only)
- Prevent deletion of audit logs
- Require encryption for all storage
- Enforce MFA for sensitive operations

### Audit Logging

**What to Log**:
- All authentication attempts (success and failure)
- All claim status changes
- All evidence uploads and validations
- All operator actions (approve, reject, request info)
- All API calls to external services
- All data access (read, write, delete)
- All configuration changes

**Log Format**:
```json
{
  "timestamp": "2024-03-20T14:22:00.123Z",
  "eventType": "CLAIM_APPROVED",
  "actor": {
    "type": "OPERATOR",
    "id": "OP-12345",
    "ip": "203.0.113.42"
  },
  "resource": {
    "type": "CLAIM",
    "id": "CLM-2024-001234"
  },
  "action": "APPROVE",
  "result": "SUCCESS",
  "metadata": {
    "approvedAmount": 37500,
    "comments": "All documents verified"
  },
  "requestId": "req-uuid-12345",
  "sessionId": "sess-uuid-67890"
}
```

**Log Storage and Retention**:
- CloudWatch Logs for real-time monitoring (30 days)
- S3 for long-term storage (7 years for compliance)
- Logs encrypted with dedicated KMS key
- Immutable logs (WORM - Write Once Read Many)
- Cross-region replication for disaster recovery

### Security Monitoring

**CloudWatch Alarms**:
- Failed authentication attempts > 10 per minute
- Unauthorized access attempts
- Unusual data access patterns
- KMS key usage anomalies
- S3 bucket policy changes
- IAM role modifications

**AWS GuardDuty**:
- Enable for threat detection
- Monitor for compromised credentials
- Detect unusual API activity
- Alert on cryptocurrency mining

**AWS Security Hub**:
- Centralized security findings
- CIS AWS Foundations Benchmark compliance
- Automated remediation for common issues

### Incident Response Plan

**Detection**:
- Automated alerts via SNS
- 24/7 monitoring dashboard
- Security team on-call rotation

**Response Procedure**:
1. Isolate affected resources
2. Preserve evidence (logs, snapshots)
3. Assess scope and impact
4. Contain the incident
5. Eradicate threat
6. Recover services
7. Post-incident review

**Notification Requirements**:
- Internal: Immediate notification to security team
- Users: Within 72 hours if PII compromised
- Regulators: As per legal requirements
- Insurance partners: If their data affected


## API Design

### REST API Endpoints

**Base URL**: `https://api.bimasathi.in/v1`

#### User Endpoints

**POST /auth/send-otp**
```json
Request:
{
  "mobile": "+919876543210",
  "language": "hi"
}

Response:
{
  "success": true,
  "message": "OTP sent successfully",
  "expiresIn": 600
}
```

**POST /auth/verify-otp**
```json
Request:
{
  "mobile": "+919876543210",
  "otp": "123456"
}

Response:
{
  "success": true,
  "sessionToken": "eyJhbGciOiJIUzI1NiIs...",
  "userId": "+919876543210",
  "expiresAt": "2024-03-21T14:22:00Z"
}
```

**GET /users/me**
```json
Headers:
Authorization: Bearer {sessionToken}

Response:
{
  "userId": "+919876543210",
  "name": "Ramesh Kumar",
  "language": "hi",
  "district": "Pune",
  "state": "Maharashtra",
  "policies": [
    {
      "policyNumber": "PMFBY-MH-2024-567890",
      "cropType": "Cotton",
      "status": "Active"
    }
  ]
}
```

#### Claim Endpoints

**POST /claims**
```json
Headers:
Authorization: Bearer {sessionToken}

Request:
{
  "policyNumber": "PMFBY-MH-2024-567890",
  "cropType": "Cotton",
  "damageType": "Flood",
  "damageDate": "2024-09-10",
  "damageDescription": "Heavy rainfall caused waterlogging",
  "areaAffected": 2.5
}

Response:
{
  "claimId": "CLM-2024-001234",
  "status": "Draft",
  "createdAt": "2024-03-20T14:22:00Z",
  "nextSteps": [
    "Upload evidence photos",
    "Review and submit claim"
  ]
}
```

**GET /claims/{claimId}**
```json
Headers:
Authorization: Bearer {sessionToken}

Response:
{
  "claimId": "CLM-2024-001234",
  "status": "Under_Review",
  "submissionDate": "2024-03-20T14:22:00Z",
  "farmerDetails": {...},
  "cropDetails": {...},
  "damageDetails": {...},
  "evidence": [...],
  "timeline": [
    {
      "date": "2024-03-20T14:22:00Z",
      "status": "Draft",
      "description": "Claim initiated"
    }
  ]
}
```

**POST /claims/{claimId}/evidence**
```json
Headers:
Authorization: Bearer {sessionToken}
Content-Type: multipart/form-data

Request:
{
  "image": <binary file>,
  "description": "Photo of flooded cotton field"
}

Response:
{
  "imageId": "IMG-001",
  "uploadedAt": "2024-03-20T14:25:00Z",
  "validationStatus": "pending",
  "message": "Image uploaded successfully. Validation in progress."
}
```

**POST /claims/{claimId}/submit**
```json
Headers:
Authorization: Bearer {sessionToken}

Request:
{
  "confirmation": true
}

Response:
{
  "claimId": "CLM-2024-001234",
  "status": "Submitted",
  "submittedAt": "2024-03-20T15:00:00Z",
  "message": "Claim submitted successfully. You will be notified of updates."
}
```

**GET /claims**
```json
Headers:
Authorization: Bearer {sessionToken}

Query Parameters:
?status=Under_Review&page=1&limit=10

Response:
{
  "claims": [
    {
      "claimId": "CLM-2024-001234",
      "status": "Under_Review",
      "submissionDate": "2024-03-20T14:22:00Z",
      "cropType": "Cotton"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 3
  }
}
```

#### Operator Endpoints

**GET /operator/claims**
```json
Headers:
Authorization: Bearer {cognitoJWT}

Query Parameters:
?status=Submitted&district=Pune&page=1&limit=20

Response:
{
  "claims": [
    {
      "claimId": "CLM-2024-001234",
      "farmerName": "Ramesh Kumar",
      "submissionDate": "2024-03-20T14:22:00Z",
      "validationScore": 88,
      "priority": "normal"
    }
  ],
  "pagination": {...}
}
```

**POST /operator/claims/{claimId}/approve**
```json
Headers:
Authorization: Bearer {cognitoJWT}

Request:
{
  "comments": "All documents verified, evidence clear",
  "approvedAmount": 37500
}

Response:
{
  "claimId": "CLM-2024-001234",
  "status": "Approved",
  "approvedAt": "2024-03-21T09:15:00Z",
  "insurerSubmissionScheduled": true
}
```

**POST /operator/claims/{claimId}/reject**
```json
Headers:
Authorization: Bearer {cognitoJWT}

Request:
{
  "reason": "Insufficient evidence",
  "requiredActions": [
    "Upload clearer photos of damaged crop",
    "Provide photos showing field landmarks"
  ]
}

Response:
{
  "claimId": "CLM-2024-001234",
  "status": "Rejected",
  "rejectedAt": "2024-03-21T09:15:00Z",
  "farmerNotified": true
}
```

**GET /operator/analytics**
```json
Headers:
Authorization: Bearer {cognitoJWT}

Query Parameters:
?fromDate=2024-03-01&toDate=2024-03-31&district=Pune

Response:
{
  "period": {
    "from": "2024-03-01",
    "to": "2024-03-31"
  },
  "metrics": {
    "totalClaims": 156,
    "approvedClaims": 136,
    "rejectedClaims": 12,
    "pendingClaims": 8,
    "approvalRate": 0.87,
    "averageProcessingTimeHours": 18.5
  },
  "byDamageType": {
    "Flood": 89,
    "Drought": 34,
    "Pest": 23,
    "Hail": 10
  },
  "byCropType": {
    "Cotton": 67,
    "Rice": 45,
    "Wheat": 32,
    "Sugarcane": 12
  }
}
```

### WebSocket API (Real-time Updates)

**Connection**: `wss://ws.bimasathi.in`

**Authentication**: Send session token in first message

**Message Types**:

**Client → Server**:
```json
{
  "type": "subscribe",
  "claimId": "CLM-2024-001234"
}
```

**Server → Client**:
```json
{
  "type": "status_update",
  "claimId": "CLM-2024-001234",
  "newStatus": "Approved",
  "timestamp": "2024-03-21T09:15:00Z",
  "message": "Your claim has been approved!"
}
```

### Error Response Format

All errors follow consistent format:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid claim data",
    "details": [
      {
        "field": "damageDate",
        "issue": "Date cannot be in the future"
      }
    ],
    "requestId": "req-uuid-12345"
  }
}
```

**Error Codes**:
- `AUTHENTICATION_REQUIRED`: 401
- `INVALID_TOKEN`: 401
- `INSUFFICIENT_PERMISSIONS`: 403
- `RESOURCE_NOT_FOUND`: 404
- `VALIDATION_ERROR`: 400
- `RATE_LIMIT_EXCEEDED`: 429
- `INTERNAL_SERVER_ERROR`: 500
- `SERVICE_UNAVAILABLE`: 503


## Error Handling Strategy

### Error Classification

**Transient Errors** (Retry with exponential backoff):
- Network timeouts
- Service throttling (429)
- Temporary service unavailability (503)
- Database connection failures
- External API timeouts

**Permanent Errors** (Fail fast, notify user):
- Authentication failures (401)
- Authorization failures (403)
- Resource not found (404)
- Validation errors (400)
- Malformed requests

**Critical Errors** (Alert operations team):
- Data corruption
- Security breaches
- Service outages
- Cascading failures

### Retry Logic

```javascript
async function retryWithBackoff(operation, maxAttempts = 3) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (!isRetryable(error) || attempt === maxAttempts) {
        throw error;
      }
      
      const delayMs = Math.min(1000 * Math.pow(2, attempt), 10000);
      await sleep(delayMs);
      
      console.log(`Retry attempt ${attempt} after ${delayMs}ms`);
    }
  }
}

function isRetryable(error) {
  const retryableStatusCodes = [408, 429, 500, 502, 503, 504];
  const retryableErrorCodes = [
    'ETIMEDOUT',
    'ECONNRESET',
    'ENOTFOUND',
    'ThrottlingException',
    'ServiceUnavailable'
  ];
  
  return retryableStatusCodes.includes(error.statusCode) ||
         retryableErrorCodes.includes(error.code);
}
```

### Circuit Breaker Pattern

```javascript
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.failureCount = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = Date.now();
  }
  
  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}

// Usage
const weatherAPICircuit = new CircuitBreaker(5, 60000);

async function getWeatherData(location, date) {
  try {
    return await weatherAPICircuit.execute(() => 
      weatherAPI.getHistoricalData(location, date)
    );
  } catch (error) {
    // Fallback: Skip weather verification
    console.warn('Weather API unavailable, skipping verification');
    return null;
  }
}
```

### Graceful Degradation

**Service Degradation Levels**:

**Level 0 - Full Service**:
- All features operational
- Real-time processing
- All validations enabled

**Level 1 - Minor Degradation**:
- Weather cross-verification disabled
- Satellite data integration disabled
- Non-critical validations skipped
- Increased processing time acceptable

**Level 2 - Moderate Degradation**:
- LLM calls use cached responses where possible
- Image analysis uses basic checks only
- Operator review required for all claims
- Batch processing instead of real-time

**Level 3 - Severe Degradation**:
- Manual claim filing via operator dashboard only
- WhatsApp/Voice interfaces show maintenance message
- Queue all requests for later processing
- Emergency contact information provided

```javascript
function getDegradationLevel() {
  const services = {
    bedrock: checkServiceHealth('bedrock'),
    rekognition: checkServiceHealth('rekognition'),
    transcribe: checkServiceHealth('transcribe'),
    dynamodb: checkServiceHealth('dynamodb'),
    s3: checkServiceHealth('s3')
  };
  
  const criticalDown = !services.dynamodb || !services.s3;
  if (criticalDown) return 3;
  
  const aiServicesDown = !services.bedrock && !services.rekognition;
  if (aiServicesDown) return 2;
  
  const someServicesDown = Object.values(services).filter(s => !s).length > 0;
  if (someServicesDown) return 1;
  
  return 0;
}
```

### User-Facing Error Messages

**Principle**: Never expose technical details to farmers

```javascript
function getUserFriendlyError(error, language) {
  const errorMessages = {
    'NETWORK_ERROR': {
      'hi': 'नेटवर्क में समस्या है। कृपया कुछ देर बाद पुनः प्रयास करें।',
      'en': 'Network issue. Please try again in a few moments.'
    },
    'VALIDATION_ERROR': {
      'hi': 'कुछ जानकारी गलत है। कृपया जांचें और फिर से भेजें।',
      'en': 'Some information is incorrect. Please check and resend.'
    },
    'SERVICE_UNAVAILABLE': {
      'hi': 'सेवा अस्थायी रूप से उपलब्ध नहीं है। हम जल्द ही वापस आएंगे।',
      'en': 'Service temporarily unavailable. We will be back soon.'
    },
    'AUTHENTICATION_FAILED': {
      'hi': 'OTP गलत है। कृपया सही OTP दर्ज करें।',
      'en': 'Incorrect OTP. Please enter the correct OTP.'
    }
  };
  
  const errorType = classifyError(error);
  return errorMessages[errorType]?.[language] || errorMessages['SERVICE_UNAVAILABLE'][language];
}
```

### Dead Letter Queues

**Failed Message Handling**:
- All async operations use SQS with DLQ
- Messages retry 3 times before moving to DLQ
- DLQ monitored with CloudWatch alarms
- Manual review and reprocessing for DLQ messages

```javascript
// SQS Queue Configuration
const queueConfig = {
  MessageRetentionPeriod: 1209600, // 14 days
  VisibilityTimeout: 300, // 5 minutes
  ReceiveMessageWaitTimeSeconds: 20, // Long polling
  RedrivePolicy: {
    deadLetterTargetArn: dlqArn,
    maxReceiveCount: 3
  }
};
```

### Monitoring and Alerting

**Error Rate Thresholds**:
- Warning: Error rate > 1% for 5 minutes
- Critical: Error rate > 5% for 5 minutes
- Emergency: Error rate > 10% for 2 minutes

**Alert Channels**:
- SNS topic for on-call engineers
- PagerDuty for critical alerts
- Slack for warnings
- Email for daily error summaries


## Monitoring and Observability

### CloudWatch Metrics

**Custom Business Metrics**:
```javascript
// Emit custom metrics
const cloudwatch = new AWS.CloudWatch();

async function emitMetric(metricName, value, unit = 'Count') {
  await cloudwatch.putMetricData({
    Namespace: 'BimaSathi',
    MetricData: [{
      MetricName: metricName,
      Value: value,
      Unit: unit,
      Timestamp: new Date(),
      Dimensions: [
        { Name: 'Environment', Value: process.env.ENVIRONMENT },
        { Name: 'Region', Value: process.env.AWS_REGION }
      ]
    }]
  }).promise();
}

// Usage examples
await emitMetric('ClaimsFiled', 1);
await emitMetric('ClaimApprovalRate', 0.87, 'Percent');
await emitMetric('EvidenceValidationScore', 88, 'None');
await emitMetric('LLMResponseTime', 1250, 'Milliseconds');
```

**Key Metrics to Track**:

**User Engagement**:
- Active users (daily, weekly, monthly)
- New user registrations
- Session duration
- Message volume
- User retention rate

**Claim Processing**:
- Claims filed per day
- Claims by status (Draft, Submitted, Approved, Rejected)
- Average claim processing time
- Approval rate
- Rejection reasons distribution

**System Performance**:
- API response time (p50, p95, p99)
- Lambda execution duration
- Lambda cold start frequency
- DynamoDB read/write capacity utilization
- S3 request rate

**AI Services**:
- Bedrock API latency
- Transcribe accuracy (estimated)
- Rekognition API calls
- LLM token usage
- Translation API calls

**Evidence Validation**:
- Average validation score
- Validation failure rate by reason
- Tamper detection rate
- Weather cross-verification match rate

**Errors and Failures**:
- Error rate by type
- Failed authentication attempts
- API 4xx and 5xx errors
- Lambda function errors
- DLQ message count

### CloudWatch Dashboards

**Operations Dashboard**:
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "title": "Claims Filed (24h)",
        "metrics": [["BimaSathi", "ClaimsFiled"]],
        "period": 300,
        "stat": "Sum"
      }
    },
    {
      "type": "metric",
      "properties": {
        "title": "API Response Time",
        "metrics": [
          ["AWS/Lambda", "Duration", {"stat": "p50"}],
          ["...", {"stat": "p95"}],
          ["...", {"stat": "p99"}]
        ]
      }
    },
    {
      "type": "metric",
      "properties": {
        "title": "Error Rate",
        "metrics": [
          ["AWS/Lambda", "Errors", {"stat": "Sum"}],
          [".", "Invocations", {"stat": "Sum"}]
        ],
        "yAxis": {"left": {"min": 0}}
      }
    }
  ]
}
```

**Business Metrics Dashboard**:
- Total claims by status (pie chart)
- Claims filed over time (line chart)
- Approval rate trend (line chart)
- Top rejection reasons (bar chart)
- Geographic distribution (map)
- Crop type distribution (pie chart)

### Distributed Tracing (AWS X-Ray)

**Instrumentation**:
```javascript
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

// Trace custom segments
exports.handler = async (event) => {
  const segment = AWSXRay.getSegment();
  
  // Trace LLM call
  const llmSubsegment = segment.addNewSubsegment('BedrockLLMCall');
  try {
    const response = await callBedrock(prompt);
    llmSubsegment.addMetadata('prompt_tokens', response.usage.prompt_tokens);
    llmSubsegment.addMetadata('completion_tokens', response.usage.completion_tokens);
    llmSubsegment.close();
    return response;
  } catch (error) {
    llmSubsegment.addError(error);
    llmSubsegment.close();
    throw error;
  }
};
```

**Trace Analysis**:
- Identify bottlenecks in claim processing workflow
- Track latency across service boundaries
- Detect anomalous patterns
- Correlate errors across services

### Logging Strategy

**Log Levels**:
- **ERROR**: System errors requiring immediate attention
- **WARN**: Potential issues, degraded functionality
- **INFO**: Important business events (claim filed, approved)
- **DEBUG**: Detailed diagnostic information (development only)

**Structured Logging**:
```javascript
const logger = {
  info: (message, context) => {
    console.log(JSON.stringify({
      level: 'INFO',
      timestamp: new Date().toISOString(),
      message,
      ...context,
      environment: process.env.ENVIRONMENT,
      requestId: context.requestId || 'unknown'
    }));
  },
  
  error: (message, error, context) => {
    console.error(JSON.stringify({
      level: 'ERROR',
      timestamp: new Date().toISOString(),
      message,
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      },
      ...context,
      environment: process.env.ENVIRONMENT
    }));
  }
};

// Usage
logger.info('Claim submitted', {
  claimId: 'CLM-2024-001234',
  userId: '+919876543210',
  cropType: 'Cotton'
});
```

**Log Aggregation**:
- CloudWatch Logs for all Lambda functions
- Log groups per function
- Retention: 30 days in CloudWatch, 7 years in S3
- CloudWatch Insights for querying

**Sample Queries**:
```sql
-- Find all failed claims in last 24 hours
fields @timestamp, claimId, error.message
| filter level = "ERROR" and eventType = "CLAIM_PROCESSING_FAILED"
| sort @timestamp desc
| limit 100

-- Calculate average claim processing time
fields @timestamp, claimId, processingTimeMs
| filter eventType = "CLAIM_APPROVED"
| stats avg(processingTimeMs) as avgTime by bin(5m)

-- Top rejection reasons
fields @timestamp, rejectionReason
| filter eventType = "CLAIM_REJECTED"
| stats count() by rejectionReason
| sort count desc
```

### Alerting Rules

**Critical Alerts** (PagerDuty):
- Error rate > 5% for 5 minutes
- API availability < 95% for 5 minutes
- DynamoDB throttling events
- Lambda function failures > 10 in 5 minutes
- Security: Failed auth attempts > 100 in 5 minutes
- Data: DLQ message count > 50

**Warning Alerts** (Slack):
- Error rate > 1% for 10 minutes
- API latency p95 > 3 seconds
- Evidence validation failure rate > 20%
- Operator review queue > 100 claims
- S3 storage > 80% of quota

**Daily Reports** (Email):
- Claims filed, approved, rejected
- User engagement metrics
- System performance summary
- Cost analysis
- Top errors and warnings

### Health Checks

**Endpoint**: `GET /health`

```json
{
  "status": "healthy",
  "timestamp": "2024-03-20T14:22:00Z",
  "version": "1.2.3",
  "services": {
    "dynamodb": {
      "status": "healthy",
      "latency": 12
    },
    "s3": {
      "status": "healthy",
      "latency": 45
    },
    "bedrock": {
      "status": "healthy",
      "latency": 1250
    },
    "rekognition": {
      "status": "healthy",
      "latency": 890
    },
    "weatherAPI": {
      "status": "degraded",
      "latency": 5000,
      "message": "High latency detected"
    }
  },
  "degradationLevel": 1
}
```


## Cost Estimation

### AWS Service Costs (Monthly Estimates)

**Assumptions**:
- 10,000 active users
- 5,000 claims filed per month
- Average 3 photos per claim (15,000 images)
- 50,000 WhatsApp messages
- 2,000 voice calls
- India (Mumbai) region pricing

**Lambda**:
- Requests: ~500,000 invocations/month
- Duration: Average 2 seconds, 512 MB memory
- Cost: $0.20 per 1M requests + $0.0000166667 per GB-second
- Estimated: $5 + $8 = **$13/month**

**DynamoDB**:
- On-demand pricing for variable workload
- Read: 1M read request units
- Write: 500K write request units
- Storage: 10 GB
- Cost: $1.25 per million reads + $1.25 per million writes + $0.25 per GB
- Estimated: $1.25 + $0.63 + $2.50 = **$4.38/month**

**S3**:
- Storage: 50 GB (evidence + documents)
- PUT requests: 15,000
- GET requests: 50,000
- Data transfer: 20 GB out
- Cost: $0.023 per GB + $0.005 per 1K PUTs + $0.0004 per 1K GETs + $0.09 per GB transfer
- Estimated: $1.15 + $0.08 + $0.02 + $1.80 = **$3.05/month**

**Amazon Bedrock** (Claude 3 Haiku):
- Input tokens: 10M tokens (~20K claims × 500 tokens)
- Output tokens: 5M tokens
- Cost: $0.25 per 1M input tokens + $1.25 per 1M output tokens
- Estimated: $2.50 + $6.25 = **$8.75/month**

**Amazon Transcribe**:
- Audio minutes: 1,000 minutes (2,000 calls × 30 seconds average)
- Cost: $0.024 per minute
- Estimated: **$24/month**

**Amazon Rekognition**:
- Images processed: 15,000
- Cost: $1.00 per 1,000 images (first 1M)
- Estimated: **$15/month**

**Amazon Translate**:
- Characters: 5M characters (claim documents)
- Cost: $15 per 1M characters
- Estimated: **$75/month**

**API Gateway**:
- Requests: 500,000
- Cost: $3.50 per million requests
- Estimated: **$1.75/month**

**CloudWatch**:
- Logs ingestion: 10 GB
- Metrics: 100 custom metrics
- Alarms: 20 alarms
- Cost: $0.50 per GB + $0.30 per metric + $0.10 per alarm
- Estimated: $5 + $30 + $2 = **$37/month**

**SNS**:
- Messages: 100,000 (notifications)
- Cost: $0.50 per million messages
- Estimated: **$0.05/month**

**EventBridge**:
- Events: 50,000
- Cost: $1.00 per million events
- Estimated: **$0.05/month**

**Step Functions**:
- State transitions: 25,000 (5,000 claims × 5 steps)
- Cost: $25 per million transitions
- Estimated: **$0.63/month**

**Cognito**:
- MAU (operators): 50
- Cost: Free tier (first 50,000 MAU)
- Estimated: **$0/month**

**KMS**:
- Keys: 3 customer-managed keys
- Requests: 100,000
- Cost: $1 per key per month + $0.03 per 10,000 requests
- Estimated: $3 + $0.30 = **$3.30/month**

**Third-Party Services**:
- WhatsApp Business API: $0.005 per message × 50,000 = **$250/month**
- Twilio Voice: $0.02 per minute × 1,000 minutes = **$20/month**
- Weather API: **$50/month** (subscription)

**Total Monthly Cost**: ~**$506/month** for 10,000 users, 5,000 claims

**Cost Per Claim**: $506 / 5,000 = **$0.10 per claim**

**Cost Per User**: $506 / 10,000 = **$0.05 per user per month**

### Cost Optimization Strategies

**Immediate Optimizations**:
1. Use Lambda reserved concurrency to avoid over-provisioning
2. Enable S3 Intelligent-Tiering for automatic cost optimization
3. Use DynamoDB on-demand for unpredictable workloads
4. Compress images before storage (reduce S3 costs by 50%)
5. Cache LLM responses for common queries (reduce Bedrock costs by 30%)
6. Use CloudWatch Logs retention policies (30 days vs. indefinite)

**Medium-Term Optimizations**:
1. Negotiate volume discounts with WhatsApp/Twilio
2. Use Savings Plans for Lambda and DynamoDB
3. Implement request batching for Rekognition
4. Use Amazon Polly instead of Transcribe where possible (cheaper)
5. Optimize LLM prompts to reduce token usage

**Scaling Projections**:

**100,000 users, 50,000 claims/month**:
- AWS services: ~$1,500/month (scales sub-linearly)
- Third-party: ~$2,500/month (scales linearly)
- Total: ~$4,000/month
- Cost per claim: **$0.08**

**1,000,000 users, 500,000 claims/month**:
- AWS services: ~$12,000/month (volume discounts)
- Third-party: ~$25,000/month (negotiated rates)
- Total: ~$37,000/month
- Cost per claim: **$0.074**


## Scalability Strategy

### Horizontal Scaling

**Serverless Auto-Scaling**:
- Lambda functions scale automatically to handle load
- No manual intervention required
- Concurrent execution limit: 1,000 (default), can request increase
- Reserved concurrency for critical functions

**DynamoDB Scaling**:
- On-demand mode for unpredictable workloads
- Auto-scaling for provisioned capacity mode
- Global tables for multi-region deployment
- DAX (DynamoDB Accelerator) for read-heavy workloads

**S3 Scaling**:
- Automatically scales to handle any request rate
- No capacity planning required
- Use CloudFront CDN for frequently accessed content
- Transfer Acceleration for faster uploads from remote locations

### Vertical Scaling

**Lambda Memory Optimization**:
```javascript
// Test different memory configurations
const memoryConfigs = [512, 1024, 1536, 2048, 3008];

// Measure execution time and cost
for (const memory of memoryConfigs) {
  const result = await testLambda(memory);
  console.log(`Memory: ${memory}MB, Duration: ${result.duration}ms, Cost: $${result.cost}`);
}

// Choose optimal configuration (usually 1024-1536 MB for our workload)
```

**Database Optimization**:
- Use DynamoDB indexes strategically
- Implement caching layer (ElastiCache) for hot data
- Batch operations where possible
- Use DynamoDB Streams for async processing

### Load Testing

**Test Scenarios**:

**Scenario 1: Normal Load**
- 100 concurrent users
- 10 claims filed per minute
- 30 images uploaded per minute
- Duration: 1 hour

**Scenario 2: Peak Load (Disaster Event)**
- 1,000 concurrent users
- 100 claims filed per minute
- 300 images uploaded per minute
- Duration: 4 hours

**Scenario 3: Sustained High Load**
- 500 concurrent users
- 50 claims filed per minute
- Duration: 24 hours

**Load Testing Tools**:
- Artillery.io for API load testing
- AWS Distributed Load Testing solution
- Custom scripts for WhatsApp/Voice simulation

**Performance Targets**:
- API response time: p95 < 2 seconds
- Image upload: p95 < 10 seconds
- Claim processing: p95 < 60 seconds
- System availability: 99.5%

### Caching Strategy

**Multi-Layer Caching**:

**Layer 1: Application Cache (Lambda)**
```javascript
// In-memory cache for Lambda warm starts
const cache = new Map();

function getCachedData(key, ttl = 300000) { // 5 minutes
  const cached = cache.get(key);
  if (cached && Date.now() - cached.timestamp < ttl) {
    return cached.data;
  }
  return null;
}

function setCachedData(key, data) {
  cache.set(key, {
    data,
    timestamp: Date.now()
  });
}
```

**Layer 2: ElastiCache (Redis)**
```javascript
const redis = require('redis');
const client = redis.createClient({
  host: process.env.REDIS_ENDPOINT,
  port: 6379
});

async function getCachedUser(userId) {
  const cached = await client.get(`user:${userId}`);
  if (cached) {
    return JSON.parse(cached);
  }
  
  const user = await dynamodb.getItem({
    TableName: 'Users',
    Key: { userId }
  });
  
  await client.setex(`user:${userId}`, 3600, JSON.stringify(user));
  return user;
}
```

**Layer 3: CloudFront CDN**
- Cache static assets (images, documents)
- Cache API responses with appropriate TTL
- Invalidate cache on updates

**Cache Invalidation**:
- Time-based expiration (TTL)
- Event-based invalidation (on data update)
- Manual invalidation for critical updates

### Database Sharding

**Sharding Strategy** (for future scale):
- Shard by state/region (geographic distribution)
- Each shard is independent DynamoDB table
- Route requests based on user's state
- Cross-shard queries via aggregation service

```javascript
function getShardForUser(userId) {
  const user = await getUserDetails(userId);
  const stateCode = user.state;
  return `bimasathi-claims-${stateCode.toLowerCase()}`;
}

async function queryClaim(claimId, userId) {
  const tableName = getShardForUser(userId);
  return await dynamodb.getItem({
    TableName: tableName,
    Key: { claimId }
  });
}
```

### Rate Limiting

**User-Level Rate Limits**:
```javascript
const rateLimiter = {
  // Max 10 claims per user per day
  claimFiling: {
    limit: 10,
    window: 86400000 // 24 hours
  },
  
  // Max 50 messages per user per hour
  messaging: {
    limit: 50,
    window: 3600000 // 1 hour
  },
  
  // Max 5 OTP requests per user per hour
  otpRequests: {
    limit: 5,
    window: 3600000
  }
};

async function checkRateLimit(userId, action) {
  const key = `ratelimit:${action}:${userId}`;
  const count = await redis.incr(key);
  
  if (count === 1) {
    await redis.expire(key, rateLimiter[action].window / 1000);
  }
  
  if (count > rateLimiter[action].limit) {
    throw new Error('Rate limit exceeded');
  }
  
  return true;
}
```

**API-Level Rate Limits**:
- API Gateway throttling: 1,000 requests per second
- Burst capacity: 2,000 requests
- Per-user quota: 10,000 requests per day

### Queue-Based Processing

**Async Processing with SQS**:
```javascript
// Producer: Add claim to processing queue
async function queueClaimProcessing(claimId) {
  await sqs.sendMessage({
    QueueUrl: process.env.CLAIM_PROCESSING_QUEUE,
    MessageBody: JSON.stringify({ claimId }),
    MessageAttributes: {
      Priority: {
        DataType: 'String',
        StringValue: 'normal'
      }
    }
  });
}

// Consumer: Process claims from queue
async function processClaimQueue() {
  const messages = await sqs.receiveMessage({
    QueueUrl: process.env.CLAIM_PROCESSING_QUEUE,
    MaxNumberOfMessages: 10,
    WaitTimeSeconds: 20 // Long polling
  });
  
  for (const message of messages.Messages || []) {
    try {
      const { claimId } = JSON.parse(message.Body);
      await processClaim(claimId);
      
      await sqs.deleteMessage({
        QueueUrl: process.env.CLAIM_PROCESSING_QUEUE,
        ReceiptHandle: message.ReceiptHandle
      });
    } catch (error) {
      console.error('Failed to process claim', error);
      // Message will be retried or moved to DLQ
    }
  }
}
```

**Priority Queues**:
- High priority: Claims near deadline
- Normal priority: Regular claims
- Low priority: Analytics, reporting


## Disaster Recovery Strategy

### Backup Strategy

**DynamoDB Backups**:
- Point-in-time recovery (PITR) enabled on all tables
- 35-day retention window
- Automated daily backups at 2 AM IST
- Cross-region backup replication to Hyderabad region
- Backup retention: 90 days

**S3 Backups**:
- Versioning enabled on all buckets
- Cross-region replication to Hyderabad region
- Lifecycle policies for cost optimization
- MFA delete protection for critical buckets

**Configuration Backups**:
- Infrastructure as Code (CloudFormation/Terraform) in Git
- Lambda function code in CodeCommit/GitHub
- Environment variables in AWS Secrets Manager
- Daily exports of all configurations

### Recovery Objectives

**RPO (Recovery Point Objective)**: 1 hour
- Maximum acceptable data loss: 1 hour of transactions
- Achieved through PITR and continuous replication

**RTO (Recovery Time Objective)**: 4 hours
- Maximum acceptable downtime: 4 hours
- Includes detection, decision, and restoration time

### Disaster Scenarios and Recovery Procedures

**Scenario 1: Single Lambda Function Failure**
- **Detection**: CloudWatch alarms, error rate spike
- **Impact**: Specific functionality unavailable
- **Recovery**: 
  1. Automatic retry by AWS Lambda
  2. If persistent, rollback to previous version
  3. Investigate and fix issue
- **RTO**: 15 minutes

**Scenario 2: DynamoDB Table Corruption**
- **Detection**: Data validation errors, user reports
- **Impact**: Incorrect data, potential data loss
- **Recovery**:
  1. Identify corruption timestamp
  2. Restore from PITR to point before corruption
  3. Replay transactions from audit logs
  4. Validate data integrity
- **RTO**: 2 hours

**Scenario 3: S3 Bucket Deletion**
- **Detection**: Immediate (MFA delete protection)
- **Impact**: Evidence and documents unavailable
- **Recovery**:
  1. Restore from cross-region replica
  2. Verify data integrity
  3. Update application to use restored bucket
- **RTO**: 1 hour

**Scenario 4: Regional Outage (Mumbai)**
- **Detection**: AWS Health Dashboard, service unavailability
- **Impact**: Complete service outage
- **Recovery**:
  1. Activate disaster recovery plan
  2. Failover to Hyderabad region
  3. Update DNS to point to DR environment
  4. Verify all services operational
  5. Communicate with users
- **RTO**: 4 hours

**Scenario 5: Complete AWS Account Compromise**
- **Detection**: Unusual activity, unauthorized access
- **Impact**: Potential data breach, service disruption
- **Recovery**:
  1. Immediately revoke all credentials
  2. Isolate affected resources
  3. Restore from backups in separate account
  4. Conduct security audit
  5. Notify affected users
- **RTO**: 8 hours

### Multi-Region Architecture

**Primary Region**: Mumbai (ap-south-1)
**DR Region**: Hyderabad (ap-south-2)

**Active-Passive Configuration**:
- Primary region handles all traffic
- DR region receives continuous data replication
- DR region infrastructure provisioned but idle
- Automated failover scripts ready

**Failover Process**:
```bash
#!/bin/bash
# Disaster Recovery Failover Script

echo "Starting DR failover to Hyderabad region..."

# 1. Update Route 53 health check
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch file://failover-dns.json

# 2. Activate DR Lambda functions
aws lambda update-function-configuration \
  --function-name bimasathi-message-handler-dr \
  --environment Variables={ACTIVE=true} \
  --region ap-south-2

# 3. Scale up DR DynamoDB tables
aws dynamodb update-table \
  --table-name bimasathi-claims-dr \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-2

# 4. Verify DR services
./verify-dr-health.sh

echo "Failover complete. DR region is now active."
```

**Failback Process**:
- Verify primary region is healthy
- Sync data from DR to primary
- Gradually shift traffic back to primary
- Deactivate DR region

### Testing and Drills

**Quarterly DR Drills**:
- Simulate regional outage
- Execute failover procedures
- Verify RTO/RPO targets met
- Document lessons learned
- Update runbooks

**Monthly Backup Restoration Tests**:
- Restore random sample of backups
- Verify data integrity
- Test restoration procedures
- Measure restoration time

**Continuous Validation**:
- Automated backup verification
- Cross-region replication monitoring
- Health checks on DR infrastructure

### Data Retention and Archival

**Retention Policies**:
- Active claims: DynamoDB (indefinite)
- Completed claims: DynamoDB (7 years), then archive to S3 Glacier
- Evidence images: S3 Standard (90 days), then S3 Glacier (7 years)
- Audit logs: CloudWatch (30 days), S3 (7 years)
- Backups: 90 days

**Archival Process**:
```javascript
// Automated archival Lambda (runs monthly)
async function archiveOldClaims() {
  const sevenYearsAgo = new Date();
  sevenYearsAgo.setFullYear(sevenYearsAgo.getFullYear() - 7);
  
  const oldClaims = await dynamodb.query({
    TableName: 'Claims',
    IndexName: 'status-submissionDate-index',
    KeyConditionExpression: 'status = :status AND submissionDate < :date',
    ExpressionAttributeValues: {
      ':status': 'Payout_Processed',
      ':date': sevenYearsAgo.toISOString()
    }
  });
  
  for (const claim of oldClaims.Items) {
    // Archive to S3 Glacier
    await s3.putObject({
      Bucket: 'bimasathi-archive',
      Key: `claims/${claim.claimId}.json`,
      Body: JSON.stringify(claim),
      StorageClass: 'GLACIER'
    });
    
    // Delete from DynamoDB
    await dynamodb.deleteItem({
      TableName: 'Claims',
      Key: { claimId: claim.claimId }
    });
  }
}
```


## Deployment Architecture

### Environment Strategy

**Environments**:
1. **Development (dev)**: Individual developer testing
2. **Staging (stage)**: Integration testing, QA
3. **Production (prod)**: Live user traffic

**Environment Isolation**:
- Separate AWS accounts per environment (recommended)
- Or separate VPCs with strict security groups
- Separate DynamoDB tables, S3 buckets, Lambda functions
- Environment-specific configuration in Secrets Manager

**Environment Configuration**:
```javascript
const config = {
  dev: {
    region: 'ap-south-1',
    dynamodbPrefix: 'bimasathi-dev',
    s3Bucket: 'bimasathi-storage-dev',
    logLevel: 'DEBUG',
    llmModel: 'anthropic.claude-3-haiku',
    whatsappWebhook: 'https://dev.bimasathi.in/webhook'
  },
  stage: {
    region: 'ap-south-1',
    dynamodbPrefix: 'bimasathi-stage',
    s3Bucket: 'bimasathi-storage-stage',
    logLevel: 'INFO',
    llmModel: 'anthropic.claude-3-haiku',
    whatsappWebhook: 'https://stage.bimasathi.in/webhook'
  },
  prod: {
    region: 'ap-south-1',
    dynamodbPrefix: 'bimasathi-prod',
    s3Bucket: 'bimasathi-storage-prod',
    logLevel: 'WARN',
    llmModel: 'anthropic.claude-3-sonnet',
    whatsappWebhook: 'https://api.bimasathi.in/webhook'
  }
};
```

### Infrastructure as Code

**CloudFormation Template Structure**:
```
infrastructure/
├── templates/
│   ├── network.yaml           # VPC, subnets, security groups
│   ├── storage.yaml            # DynamoDB tables, S3 buckets
│   ├── compute.yaml            # Lambda functions
│   ├── api.yaml                # API Gateway
│   ├── orchestration.yaml      # Step Functions, EventBridge
│   ├── monitoring.yaml         # CloudWatch, alarms
│   └── security.yaml           # IAM roles, KMS keys
├── parameters/
│   ├── dev.json
│   ├── stage.json
│   └── prod.json
└── deploy.sh
```

**Sample CloudFormation Template** (Lambda Function):
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: BimaSathi Lambda Functions

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, stage, prod]
  
Resources:
  MessageHandlerFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub bimasathi-message-handler-${Environment}
      Runtime: nodejs18.x
      Handler: index.handler
      Role: !GetAtt MessageHandlerRole.Arn
      Code:
        S3Bucket: !Sub bimasathi-deployments-${Environment}
        S3Key: message-handler.zip
      Environment:
        Variables:
          ENVIRONMENT: !Ref Environment
          DYNAMODB_TABLE: !Sub bimasathi-sessions-${Environment}
          BEDROCK_MODEL: anthropic.claude-3-haiku-20240307-v1:0
      MemorySize: 1024
      Timeout: 30
      ReservedConcurrentExecutions: 100
      
  MessageHandlerRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: DynamoDBAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                  - dynamodb:UpdateItem
                Resource: !Sub arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/bimasathi-*-${Environment}
```

### CI/CD Pipeline

**Pipeline Stages**:

1. **Source**: Code commit to GitHub/CodeCommit
2. **Build**: Compile, run tests, package artifacts
3. **Test**: Unit tests, integration tests, security scans
4. **Deploy to Dev**: Automatic deployment
5. **Deploy to Stage**: Automatic deployment after dev success
6. **Manual Approval**: Human review before production
7. **Deploy to Prod**: Blue-green deployment
8. **Smoke Tests**: Verify production deployment
9. **Rollback**: Automatic rollback on failure

**GitHub Actions Workflow**:
```yaml
name: Deploy BimaSathi

on:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm test
      - run: npm run lint
      
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  
  deploy-dev:
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - name: Deploy to Dev
        run: |
          npm run build
          aws cloudformation deploy \
            --template-file infrastructure/templates/main.yaml \
            --stack-name bimasathi-dev \
            --parameter-overrides file://infrastructure/parameters/dev.json \
            --capabilities CAPABILITY_IAM
  
  deploy-stage:
    needs: deploy-dev
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - name: Deploy to Stage
        run: |
          npm run build
          aws cloudformation deploy \
            --template-file infrastructure/templates/main.yaml \
            --stack-name bimasathi-stage \
            --parameter-overrides file://infrastructure/parameters/stage.json \
            --capabilities CAPABILITY_IAM
  
  deploy-prod:
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.bimasathi.in
    steps:
      - uses: actions/checkout@v3
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ap-south-1
      - name: Deploy to Production
        run: |
          npm run build
          aws cloudformation deploy \
            --template-file infrastructure/templates/main.yaml \
            --stack-name bimasathi-prod \
            --parameter-overrides file://infrastructure/parameters/prod.json \
            --capabilities CAPABILITY_IAM
      - name: Run smoke tests
        run: npm run smoke-test:prod
```

### Blue-Green Deployment

**Strategy**:
- Deploy new version alongside existing version
- Route small percentage of traffic to new version (canary)
- Monitor metrics and errors
- Gradually increase traffic to new version
- Rollback if issues detected

**Lambda Alias Configuration**:
```javascript
// Create new version
const newVersion = await lambda.publishVersion({
  FunctionName: 'bimasathi-message-handler-prod'
}).promise();

// Update alias to route 10% traffic to new version
await lambda.updateAlias({
  FunctionName: 'bimasathi-message-handler-prod',
  Name: 'live',
  FunctionVersion: newVersion.Version,
  RoutingConfig: {
    AdditionalVersionWeights: {
      [newVersion.Version]: 0.1
    }
  }
}).promise();

// Monitor for 30 minutes
await sleep(30 * 60 * 1000);

// If successful, route 100% traffic
await lambda.updateAlias({
  FunctionName: 'bimasathi-message-handler-prod',
  Name: 'live',
  FunctionVersion: newVersion.Version
}).promise();
```

### Rollback Procedures

**Automatic Rollback Triggers**:
- Error rate > 5% for 5 minutes
- API latency p95 > 5 seconds
- Failed health checks
- CloudWatch alarm breach

**Manual Rollback**:
```bash
# Rollback Lambda function to previous version
aws lambda update-alias \
  --function-name bimasathi-message-handler-prod \
  --name live \
  --function-version $PREVIOUS_VERSION

# Rollback CloudFormation stack
aws cloudformation update-stack \
  --stack-name bimasathi-prod \
  --use-previous-template \
  --parameters UsePreviousValue=true

# Verify rollback
./verify-deployment.sh
```

### Deployment Checklist

**Pre-Deployment**:
- [ ] All tests passing
- [ ] Security scan completed
- [ ] Code review approved
- [ ] Database migrations tested
- [ ] Rollback plan documented
- [ ] Stakeholders notified

**During Deployment**:
- [ ] Monitor CloudWatch metrics
- [ ] Check error logs
- [ ] Verify health checks
- [ ] Test critical user flows
- [ ] Monitor user feedback channels

**Post-Deployment**:
- [ ] Smoke tests passed
- [ ] Performance metrics normal
- [ ] No increase in error rate
- [ ] User feedback positive
- [ ] Documentation updated
- [ ] Deployment retrospective scheduled


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Voice and Transcription Properties

**Property 1: Voice Transcription Completeness**
*For any* voice input in a supported language (Hindi, Marathi, Telugu, Tamil, Kannada, Gujarati, Punjabi, Bengali), the Voice_Interface should successfully transcribe the audio to text using Amazon Transcribe.
**Validates: Requirements 1.1, 1.3**

**Property 2: Structured Data Extraction from Transcription**
*For any* transcribed text containing claim information, the LLM should extract all present structured fields (farmer name, location, crop type, damage description) and return them in a consistent format.
**Validates: Requirements 1.2**

**Property 3: Extraction Confirmation Round-Trip**
*For any* extracted structured data, when the system reads back the information to the farmer and receives confirmation, a new claim record should be created in DynamoDB with matching data.
**Validates: Requirements 1.5, 1.7**

### WhatsApp Interface Properties

**Property 4: Language Persistence**
*For any* farmer who selects a language preference, all subsequent system interactions should be conducted in that selected language until explicitly changed.
**Validates: Requirements 2.2, 9.2**

**Property 5: Conversational Flow Completeness**
*For any* new claim initiation, the system should collect all required information fields (policy number, crop type, damage type, damage date, location) through the conversational flow before allowing submission.
**Validates: Requirements 2.3**

**Property 6: Information Completeness Validation**
*For any* farmer-provided information during claim filing, if required fields are missing, the LLM should identify the missing fields and ask specific follow-up questions.
**Validates: Requirements 2.4**

**Property 7: Image Format and Size Validation**
*For any* uploaded image, the WhatsApp_Interface should accept it if and only if it is in JPEG, PNG, or HEIC format and is 16MB or smaller.
**Validates: Requirements 2.5**

**Property 8: Mixed Language Intent Understanding**
*For any* farmer input in mixed language or using voice-to-text, the LLM should correctly identify the intent and respond appropriately in the farmer's preferred language.
**Validates: Requirements 2.7**

### Evidence Validation Properties

**Property 9: EXIF Metadata Extraction**
*For any* uploaded photo containing EXIF data, the Evidence_Intelligence should extract GPS coordinates, timestamp, and device information.
**Validates: Requirements 3.2**

**Property 10: Geospatial Validation**
*For any* photo with GPS coordinates, the system should calculate the distance to the farmer's registered land location and flag the photo if the distance exceeds 500 meters.
**Validates: Requirements 3.3**

**Property 11: Temporal Validation**
*For any* photo with a timestamp, the system should verify that the timestamp falls within the valid claim filing window for the associated policy.
**Validates: Requirements 3.4**

**Property 12: Image Content Analysis**
*For any* uploaded photo, Rekognition should analyze the image and return labels indicating detected objects, crop types, and environmental conditions.
**Validates: Requirements 3.6**

**Property 13: Validation Feedback Generation**
*For any* image analysis that detects potential issues (poor lighting, crop not visible, etc.), the system should generate specific, actionable feedback for the farmer.
**Validates: Requirements 3.7**

**Property 14: Weather Cross-Verification**
*For any* complete evidence set with location and date information, the Evidence_Intelligence should query the Weather_API and compare the weather data with the claimed damage type.
**Validates: Requirements 3.8**

**Property 15: Weather Contradiction Flagging**
*For any* claim where weather data contradicts the claimed damage type (e.g., no rainfall for flood claim), the system should flag the claim for operator review.
**Validates: Requirements 3.9**

**Property 16: Validation Score Generation**
*For any* completed evidence validation, the system should generate a numerical validation score (0-100) and confidence level based on all validation checks.
**Validates: Requirements 3.10**

### Claim Draft Generation Properties

**Property 17: Claim Document Generation**
*For any* complete set of claim information (farmer details, policy, crop, damage, evidence), the LLM should generate a structured claim document following the insurance provider's format.
**Validates: Requirements 4.1**

**Property 18: Mandatory Field Population**
*For any* generated claim draft, all mandatory fields (policy number, farmer details, crop information, damage assessment, evidence references) should be populated with non-null values.
**Validates: Requirements 4.2**

**Property 19: Multilingual Document Generation**
*For any* claim draft, the system should generate both a version in the farmer's preferred language and an English version for official submission.
**Validates: Requirements 4.3, 4.4**

**Property 20: Draft Preview Delivery**
*For any* ready claim draft, the system should send a preview to the farmer via WhatsApp with key details highlighted.
**Validates: Requirements 4.5**

**Property 21: PDF Generation with Required Elements**
*For any* farmer-approved draft, the system should generate a PDF document that includes a unique claim reference number, QR code for verification, and digital signature placeholder.
**Validates: Requirements 4.7, 4.8**

**Property 22: Encrypted PDF Storage**
*For any* generated PDF document, the system should store it in S3 with encryption at rest enabled.
**Validates: Requirements 4.9**

### Deadline and Reminder Properties

**Property 23: Deadline Calculation**
*For any* registered policy, the Deadline_Engine should calculate all critical dates (damage reporting, claim filing, document submission) based on the policy terms.
**Validates: Requirements 5.1**

**Property 24: Multi-Stage Reminder Delivery**
*For any* approaching deadline, the system should send reminders at 7 days before, 3 days before, and 1 day before the deadline date.
**Validates: Requirements 5.2**

**Property 25: Multi-Channel Reminder Fallback**
*For any* reminder, the system should attempt delivery via WhatsApp first, then SMS if WhatsApp fails, then voice call if SMS fails.
**Validates: Requirements 5.3**

**Property 26: Incomplete Claim Nudge**
*For any* claim that is started but not completed within 24 hours, the system should send a gentle nudge to the farmer.
**Validates: Requirements 5.4**

**Property 27: Operator Reminder for Pending Claims**
*For any* claim submitted but pending operator approval for more than 48 hours, the system should send a reminder to the assigned operator.
**Validates: Requirements 5.5**

**Property 28: Missed Deadline Notification**
*For any* deadline that passes without claim submission, the system should notify the farmer and provide escalation options.
**Validates: Requirements 5.6**

**Property 29: Independent Multi-Policy Deadline Tracking**
*For any* farmer with multiple policies, the system should track deadlines independently for each policy without interference.
**Validates: Requirements 5.8**

### Status Tracking Properties

**Property 30: Status Query Response**
*For any* farmer status query (via "status" keyword or equivalent), the system should retrieve and display all claims associated with that farmer's account.
**Validates: Requirements 6.1**

**Property 31: Status Display Completeness**
*For any* displayed claim status, the system should include claim reference number, submission date, current stage, and expected timeline.
**Validates: Requirements 6.2**

**Property 32: Proactive Status Change Notification**
*For any* claim status change, the system should proactively send a notification to the farmer via WhatsApp without requiring a query.
**Validates: Requirements 6.3**

**Property 33: Specific Claim Detail Retrieval**
*For any* valid claim reference number query, the system should return detailed status including operator comments and pending actions.
**Validates: Requirements 6.4**

**Property 34: Rejection Reason Provision**
*For any* rejected claim, the system should provide specific rejection reasons and guidance on how to resubmit.
**Validates: Requirements 6.6**

**Property 35: Payout Notification with Details**
*For any* claim with payout processed status, the system should notify the farmer with the payout amount and expected credit date.
**Validates: Requirements 6.7**

### Operator Dashboard Properties

**Property 36: Jurisdiction-Based Claim Display**
*For any* operator login, the Operator_Dashboard should display only claims assigned to that operator's jurisdiction (district/state).
**Validates: Requirements 7.1**

**Property 37: Claim List Field Completeness**
*For any* claim displayed in the operator's list view, the system should show claim reference, farmer name, submission date, validation score, and priority.
**Validates: Requirements 7.2**

**Property 38: Full Claim Detail Display**
*For any* operator-selected claim, the system should display complete claim details including all evidence photos, extracted data, validation results, and weather cross-verification.
**Validates: Requirements 7.3**

**Property 39: Claim Approval Workflow**
*For any* operator-approved claim, the system should update the claim status to "Approved" and trigger submission to the Insurance_Provider.
**Validates: Requirements 7.5**

**Property 40: Claim Rejection with Reason**
*For any* operator-rejected claim, the system should require a rejection reason and automatically notify the farmer with specific feedback.
**Validates: Requirements 7.6**

**Property 41: Additional Information Request**
*For any* operator request for additional information, the system should send a structured request to the farmer via WhatsApp.
**Validates: Requirements 7.7**

**Property 42: Dashboard Filtering**
*For any* applied filter (status, date, crop type, validation score), the dashboard should display only claims matching all filter criteria.
**Validates: Requirements 7.8**

**Property 43: Concurrent Edit Prevention**
*For any* claim being edited by an operator, the system should prevent other operators from simultaneously editing the same claim.
**Validates: Requirements 7.9**

**Property 44: Statistics Calculation**
*For any* selected time period (daily, weekly, monthly), the dashboard should calculate and display statistics including claims processed, approval rate, and average processing time.
**Validates: Requirements 7.10**

### Security and Authentication Properties

**Property 45: OTP Verification for Registration**
*For any* new farmer registration, the system should send an OTP to the provided mobile number and create a user profile only after successful OTP verification.
**Validates: Requirements 8.1, 8.2**

**Property 46: Session Verification for Claims**
*For any* claim initiation, the system should verify that the session is associated with a registered and authenticated mobile number.
**Validates: Requirements 8.4**

**Property 47: Suspicious Activity Re-Authentication**
*For any* detected suspicious activity, the system should require re-authentication via OTP before allowing further actions.
**Validates: Requirements 8.5**

**Property 48: Rate Limiting Enforcement**
*For any* policy, the system should enforce a maximum of 5 claim submissions per season and reject additional submissions beyond this limit.
**Validates: Requirements 8.6**

**Property 49: PII Encryption at Rest**
*For any* stored personally identifiable information, the system should encrypt the data at rest using AWS KMS.
**Validates: Requirements 8.7**

**Property 50: Audit Log Creation**
*For any* user action, operator action, or system event, the system should create an audit log entry in CloudWatch Logs with timestamp, actor, and action details.
**Validates: Requirements 8.9**

### Localization Properties

**Property 51: Date and Number Formatting**
*For any* displayed date or number, the system should format it according to Indian conventions (DD/MM/YYYY for dates, lakhs/crores for numbers).
**Validates: Requirements 9.5**

**Property 52: Language Switch Continuity**
*For any* farmer who switches language mid-conversation, all subsequent messages should be in the newly selected language.
**Validates: Requirements 9.7**

### Offline and Bandwidth Properties

**Property 53: Image Compression**
*For any* photo upload, the system should compress the image to reduce bandwidth while maintaining sufficient quality for evidence validation.
**Validates: Requirements 10.2**

**Property 54: Message Size Optimization**
*For any* system-generated message, the size should be optimized to work efficiently on 2G networks (typically under 1KB for text messages).
**Validates: Requirements 10.5**

### Tamper Detection Properties

**Property 55: EXIF Tamper Analysis**
*For any* uploaded photo, the Evidence_Intelligence should analyze EXIF metadata for signs of editing software usage or manipulation.
**Validates: Requirements 11.1**

**Property 56: Editing Software Flagging**
*For any* photo whose metadata indicates editing software was used, the system should flag the evidence for manual operator review.
**Validates: Requirements 11.2**

**Property 57: GPS Spoofing Detection**
*For any* set of photos with GPS coordinates that show impossible movement patterns or inconsistencies, the system should flag potential GPS spoofing.
**Validates: Requirements 11.3**

**Property 58: Timestamp Discrepancy Flagging**
*For any* photo where the EXIF timestamp differs significantly from the upload timestamp without valid explanation, the system should flag the discrepancy.
**Validates: Requirements 11.4**

**Property 59: Tampering Probability Score**
*For any* photo analyzed by Rekognition, the system should calculate a tampering probability score based on detected manipulation artifacts.
**Validates: Requirements 11.5**

**Property 60: Multi-Photo Consistency Check**
*For any* set of multiple photos in a claim, if lighting or weather conditions are inconsistent across photos, the system should flag for cross-verification.
**Validates: Requirements 11.6**

**Property 61: Flagged Claim Routing**
*For any* claim flagged by tamper detection, the system should route it to operator review rather than automatically rejecting it.
**Validates: Requirements 11.7**

### Insurance Provider Integration Properties

**Property 62: Claim Data Formatting for Insurer**
*For any* operator-approved claim, the system should format the claim data according to the specific Insurance_Provider's API specification.
**Validates: Requirements 12.1**

**Property 63: Submission Data Completeness**
*For any* claim submitted to an insurer, the system should include all required fields, evidence URLs, and validation metadata.
**Validates: Requirements 12.2**

**Property 64: API Retry with Exponential Backoff**
*For any* failed Insurance_Provider API call, the system should retry with exponential backoff for up to 24 hours before marking as failed.
**Validates: Requirements 12.3**

**Property 65: Acknowledgment Reference Storage**
*For any* successful insurer submission, the system should store the insurer's acknowledgment reference number in the claim record.
**Validates: Requirements 12.4**

**Property 66: Multi-Method Submission Support**
*For any* claim, the system should support submission via either API-based integration or email-based submission depending on insurer configuration.
**Validates: Requirements 12.6**

**Property 67: Email Submission Format**
*For any* email-based claim submission, the system should send to configured email addresses with standardized subject line and attachment naming convention.
**Validates: Requirements 12.7**

### Helper Mode Properties

**Property 68: Assisted Filing Farmer Details Collection**
*For any* helper filing on behalf of another farmer, the system should collect the farmer's mobile number and policy details before proceeding.
**Validates: Requirements 13.2**

**Property 69: Consent OTP for Assisted Filing**
*For any* assisted filing, the system should send an OTP to the farmer's mobile number and require confirmation before authorizing the helper.
**Validates: Requirements 13.3, 13.4**

**Property 70: Dual Approval Notification**
*For any* helper-completed claim that receives farmer approval, the system should send confirmation to both the farmer and the helper.
**Validates: Requirements 13.6**

**Property 71: Assisted Filing Audit Trail**
*For any* claim filed by a helper, the system should maintain a record of who filed the claim and on whose behalf for audit purposes.
**Validates: Requirements 13.7**

**Property 72: Helper Profile Switching**
*For any* helper assisting multiple farmers, the system should allow switching between farmer profiles and maintain separate contexts for each.
**Validates: Requirements 13.8**

### Analytics and Reporting Properties

**Property 73: Aggregate Statistics Display**
*For any* operator dashboard view, the system should display aggregate statistics including total claims, approval rate, rejection reasons, and average processing time.
**Validates: Requirements 14.1**

**Property 74: Analytics Filtering**
*For any* applied analytics filter (date range, geography, crop type, damage type), the system should return only data matching all filter criteria.
**Validates: Requirements 14.2**

**Property 75: Visualization Generation**
*For any* analytics data set, the system should generate appropriate visualizations including trend charts, geographic heat maps, and category breakdowns.
**Validates: Requirements 14.3**

**Property 76: Rejection Pattern Detection**
*For any* emerging pattern in rejection reasons, the system should highlight common issues and suggest interventions.
**Validates: Requirements 14.4**

**Property 77: User Engagement Tracking**
*For any* user activity (login, claim filed, message sent), the system should track engagement metrics including active users, claims per user, and completion rates.
**Validates: Requirements 14.5**

**Property 78: Validation Failure Pattern Flagging**
*For any* evidence validation criterion that fails frequently, the system should flag it for process improvement review.
**Validates: Requirements 14.6**

**Property 79: Report Export in Multiple Formats**
*For any* generated report, the system should support export in both CSV and PDF formats.
**Validates: Requirements 14.7**

**Property 80: Anonymized API Data Access**
*For any* API request for aggregate data, the system should return anonymized data with all personally identifiable information removed.
**Validates: Requirements 14.8**

### Consent and Privacy Properties

**Property 81: First-Use Consent Form Display**
*For any* farmer using the system for the first time, the system should present a clear consent form in their selected language before allowing any data collection.
**Validates: Requirements 15.1**

**Property 82: Granular Consent Collection**
*For any* consent collection, the system should separately request permission for personal data collection, location data collection, and data sharing with insurers.
**Validates: Requirements 15.2**

**Property 83: Consent Metadata Recording**
*For any* provided consent, the system should record the consent timestamp, version number, and scope in DynamoDB.
**Validates: Requirements 15.3**

**Property 84: Consent Withdrawal Processing**
*For any* farmer consent withdrawal request, the system should stop processing new claims while retaining existing claims for legal compliance.
**Validates: Requirements 15.4, 15.5**

**Property 85: Data Portability**
*For any* farmer request for their data, the system should provide the data in a portable format within 30 days.
**Validates: Requirements 15.6**

**Property 86: Data Deletion with Legal Compliance**
*For any* farmer request for data deletion, the system should delete the data subject to legal retention requirements.
**Validates: Requirements 15.7**

**Property 87: Consent-Based Data Sharing**
*For any* data sharing with third parties, the system should only share data for which the farmer has explicitly provided consent.
**Validates: Requirements 15.8**

**Property 88: Privacy Policy Accessibility**
*For any* farmer request for the privacy policy (via WhatsApp command or web URL), the system should provide access to the current privacy policy.
**Validates: Requirements 15.9**


## Testing Strategy

### Dual Testing Approach

BimaSathi requires both unit testing and property-based testing to ensure comprehensive correctness. These approaches are complementary:

- **Unit tests** verify specific examples, edge cases, and error conditions
- **Property tests** verify universal properties across all inputs
- Together, they provide comprehensive coverage: unit tests catch concrete bugs, property tests verify general correctness

### Unit Testing Strategy

Unit tests should focus on:

**Specific Examples**:
- Test that a valid Hindi voice message "मेरी कपास की फसल बाढ़ में बर्बाद हो गई" correctly extracts crop type "Cotton" and damage type "Flood"
- Test that a photo taken on 2024-09-10 for a claim with deadline 2024-09-17 passes timestamp validation
- Test that GPS coordinates (18.6298, 73.9925) within 500m of registered location (18.6300, 73.9920) pass geospatial validation

**Edge Cases**:
- Test that a photo without EXIF data triggers appropriate error message
- Test that a claim with 6 submissions (exceeding rate limit of 5) is rejected
- Test that a farmer in an area with no connectivity receives guidance to visit CSC
- Test that poor quality voice input triggers re-recording prompt

**Integration Points**:
- Test WhatsApp webhook integration with mock payloads
- Test Bedrock LLM integration with sample prompts
- Test Rekognition integration with test images
- Test Weather API integration with mock responses
- Test DynamoDB read/write operations
- Test S3 upload/download operations

**Error Conditions**:
- Test behavior when Bedrock API times out
- Test behavior when DynamoDB is throttled
- Test behavior when S3 upload fails
- Test behavior when Weather API is unavailable
- Test behavior when invalid OTP is provided
- Test behavior when claim reference number doesn't exist

### Property-Based Testing Strategy

Property-based testing is the primary method for validating the 88 correctness properties defined in this design. Each property should be implemented as a property-based test.

**Property-Based Testing Library**:
- **For Python**: Use Hypothesis
- **For TypeScript/JavaScript**: Use fast-check
- **For Java**: Use jqwik or QuickCheck for Java

**Test Configuration**:
- Each property test MUST run minimum 100 iterations (due to randomization)
- Each property test MUST be tagged with a comment referencing the design property
- Tag format: `// Feature: bimasathi, Property {number}: {property_text}`

**Example Property Test (TypeScript with fast-check)**:
```typescript
import fc from 'fast-check';

// Feature: bimasathi, Property 10: Geospatial Validation
// For any photo with GPS coordinates, the system should calculate the distance
// to the farmer's registered land location and flag if distance exceeds 500 meters
test('Property 10: Geospatial Validation', () => {
  fc.assert(
    fc.property(
      fc.record({
        photoGPS: fc.record({
          lat: fc.double({ min: -90, max: 90 }),
          lon: fc.double({ min: -180, max: 180 })
        }),
        registeredGPS: fc.record({
          lat: fc.double({ min: -90, max: 90 }),
          lon: fc.double({ min: -180, max: 180 })
        })
      }),
      (data) => {
        const distance = calculateDistance(data.photoGPS, data.registeredGPS);
        const result = validateGeolocation(data.photoGPS, data.registeredGPS);
        
        if (distance > 500) {
          expect(result.flagged).toBe(true);
          expect(result.reason).toContain('location_mismatch');
        } else {
          expect(result.flagged).toBe(false);
        }
      }
    ),
    { numRuns: 100 }
  );
});
```

**Example Property Test (Python with Hypothesis)**:
```python
from hypothesis import given, strategies as st
import pytest

# Feature: bimasathi, Property 23: Deadline Calculation
# For any registered policy, the Deadline_Engine should calculate all critical dates
@given(
    policy=st.fixed_dictionaries({
        'policyNumber': st.text(min_size=10, max_size=20),
        'startDate': st.datetimes(),
        'cropType': st.sampled_from(['Rice', 'Wheat', 'Cotton', 'Sugarcane']),
        'damageReportingWindow': st.integers(min_value=24, max_value=168),  # hours
        'claimFilingWindow': st.integers(min_value=7, max_value=30),  # days
        'documentSubmissionWindow': st.integers(min_value=7, max_value=30)  # days
    })
)
def test_property_23_deadline_calculation(policy):
    """
    Feature: bimasathi, Property 23: Deadline Calculation
    For any registered policy, the Deadline_Engine should calculate all critical dates
    """
    deadlines = calculate_deadlines(policy)
    
    # All deadlines should be present
    assert 'damageReportingDeadline' in deadlines
    assert 'claimFilingDeadline' in deadlines
    assert 'documentSubmissionDeadline' in deadlines
    
    # Deadlines should be in chronological order
    assert deadlines['damageReportingDeadline'] < deadlines['claimFilingDeadline']
    assert deadlines['claimFilingDeadline'] <= deadlines['documentSubmissionDeadline']
    
    # Deadlines should be after policy start date
    assert deadlines['damageReportingDeadline'] > policy['startDate']
```

**Generator Strategies**:

For effective property-based testing, define custom generators for domain objects:

```typescript
// Custom generators for BimaSathi domain objects
const claimDataGenerator = fc.record({
  farmerId: fc.string({ minLength: 10, maxLength: 15 }),
  policyNumber: fc.string({ minLength: 15, maxLength: 25 }),
  cropType: fc.constantFrom('Rice', 'Wheat', 'Cotton', 'Sugarcane', 'Maize'),
  damageType: fc.constantFrom('Flood', 'Drought', 'Pest', 'Hail', 'Wind'),
  damageDate: fc.date({ min: new Date('2024-01-01'), max: new Date('2024-12-31') }),
  damageDescription: fc.string({ minLength: 20, maxLength: 500 }),
  areaAffected: fc.double({ min: 0.1, max: 50 }),
  location: fc.record({
    lat: fc.double({ min: 8.0, max: 37.0 }),  // India latitude range
    lon: fc.double({ min: 68.0, max: 97.0 })  // India longitude range
  })
});

const evidencePhotoGenerator = fc.record({
  imageId: fc.uuid(),
  s3Key: fc.string({ minLength: 20, maxLength: 100 }),
  uploadTimestamp: fc.date(),
  metadata: fc.record({
    gpsCoordinates: fc.record({
      lat: fc.double({ min: 8.0, max: 37.0 }),
      lon: fc.double({ min: 68.0, max: 97.0 })
    }),
    captureTimestamp: fc.date(),
    deviceModel: fc.constantFrom('Samsung Galaxy M12', 'Redmi Note 10', 'iPhone 12', 'OnePlus Nord'),
    editingSoftware: fc.option(fc.constantFrom('Photoshop', 'GIMP', 'Snapseed'), { nil: null })
  })
});
```

### Test Coverage Requirements

- **Code Coverage**: Minimum 80% line coverage for all Lambda functions
- **Property Coverage**: All 88 correctness properties must have corresponding property-based tests
- **Integration Coverage**: All external service integrations must have integration tests
- **End-to-End Coverage**: Critical user journeys must have E2E tests

### Test Execution Strategy

**Local Development**:
- Run unit tests on every code change
- Run property tests before committing
- Use test doubles for external services

**CI/CD Pipeline**:
- Run all unit tests on every pull request
- Run all property tests on every pull request
- Run integration tests on merge to develop
- Run E2E tests before production deployment

**Production Monitoring**:
- Synthetic tests running every 5 minutes
- Monitor property violations in production logs
- Alert on unexpected property failures

### Test Data Management

**Synthetic Test Data**:
- Generate realistic farmer profiles with Indian names, locations
- Generate realistic crop and damage scenarios
- Generate realistic evidence photos (or use test image set)
- Ensure test data covers all supported languages

**Test Data Privacy**:
- Never use real farmer data in tests
- Anonymize any production data used for testing
- Ensure test data complies with data protection regulations

### Performance Testing

**Load Testing**:
- Simulate 10,000 concurrent users
- Test claim filing under peak load (100 claims/minute)
- Test evidence upload under load (300 images/minute)
- Measure response times at various load levels

**Stress Testing**:
- Test system behavior at 2x expected load
- Test graceful degradation under extreme load
- Test recovery after load spike

**Endurance Testing**:
- Run system under normal load for 24 hours
- Monitor for memory leaks
- Monitor for performance degradation over time

