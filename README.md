# 🌾 BimaSathi

### AI-Powered, Voice-First Crop Insurance Claims for Indian Farmers

> *What UPI did for payments, BimaSathi does for crop insurance claims.*

---

## 🚨 The Problem

In India, thousands of farmers lose crop insurance payouts every year — not because damage didn’t occur, but because:

* Claims are filed late
* Documentation is incomplete
* Evidence is poorly captured
* Forms are complex and English-heavy
* Deadlines are missed
* There is no structured follow-up

The system is paperwork-heavy, literacy-dependent, and not designed for rural accessibility.

---

## 💡 Our Solution

**BimaSathi** is a **WhatsApp + Voice-first multilingual AI assistant** that enables farmers to file crop insurance claims in minutes — securely, compliantly, and confidently.

No app installation.
No complicated forms.
No English barrier.

Just voice or WhatsApp.

---

## 🎯 Key Features

### 📱 WhatsApp-Based Claim Filing

* Guided conversational flow
* No new app required
* Works on low-end smartphones

### 🎙️ Voice-First Regional Language Support

* Farmers speak in their own language
* AI converts speech to structured claim data
* Designed for low literacy users

### 📸 Evidence Intelligence Engine

* Geo-tagged image validation
* Timestamp verification
* Weather cross-check using APIs
* AI-based image inspection (damage detection ready)
* Tamper-resistance checks

### 🧾 AI Claim Draft Generation

* Auto-fills insurance-compliant forms
* Multi-language support
* PDF generation
* Pre-submission validation

### ⏰ Deadline Tracking & Follow-Ups

* Automatic claim deadline detection
* WhatsApp/SMS reminders
* Status tracking until payout

### 🖥️ Operator Dashboard

* Review submitted claims
* Flag incomplete submissions
* Analytics & monitoring
* CSC / NGO support mode

### 🔐 Security & Trust by Design

* OTP verification
* Consent capture
* Encrypted storage
* Audit logs
* Role-based access control

---

## 🏗️ Architecture Overview

BimaSathi is built using a **serverless, scalable AWS-first architecture**.

### User Access Layer

* WhatsApp Business API
* Twilio
* Voice / IVR
* SMS fallback

### AI & Intelligence Layer

* Amazon Bedrock (LLM processing)
* Amazon Rekognition (image validation)
* Amazon Transcribe (speech-to-text)
* Weather APIs
* Satellite APIs (optional)

### Backend & Workflow

* AWS Lambda
* AWS Step Functions
* AWS EventBridge
* Amazon SNS

### Storage

* Amazon S3 (images & documents)
* Amazon DynamoDB (claim data & metadata)

### Security

* AWS Cognito
* Twilio Verify
* IAM role-based access
* Encryption at rest & in transit

---

## 🔄 How It Works

1. Farmer initiates claim via WhatsApp or Voice.
2. AI collects basic details conversationally.
3. Farmer uploads crop damage photos.
4. Evidence engine validates:

   * Location
   * Timestamp
   * Weather consistency
5. AI generates structured claim draft.
6. Farmer confirms submission.
7. PDF is generated.
8. Claim is routed to operator/insurer.
9. Status tracking & reminders continue until payout.

---

## 🛣️ Roadmap

### Phase 1 – MVP

* WhatsApp claim filing
* Image uploads
* Basic evidence validation
* Claim draft generation
* Reminder engine

### Phase 2 – AI Enhancements

* Damage classification models
* High-fidelity multilingual voice
* Insurance portal integrations
* Auto status polling

### Phase 3 – Scale & Governance

* Fraud detection AI
* Analytics dashboard
* Multi-state rollout
* Full RBAC governance

---

## 📊 Impact Vision

* Reduce claim rejection due to documentation errors
* Increase claim filing success rate
* Improve financial security for small farmers
* Enable inclusive digital access in rural India
* Build trust in agri-insurance systems

---

## 🎯 Target Users

* Small & marginal farmers
* Low-literacy rural populations
* Farmer families & helpers
* NGOs / CSC operators
* Insurance providers

---

## 🧠 Why This Matters

India’s farmers face climate uncertainty, crop loss, and financial instability. Insurance is meant to protect them — but complexity prevents access.

BimaSathi bridges that gap using AI built for Bharat.

This is not just automation.
This is accessibility infrastructure.

---

## 🏆 Hackathon Context

Built for: **AWS AI for Bharat Hackathon**
Problem Statement: *AI for Communities, Access & Public Impact*

---

## 🔧 Setup (High-Level)

```bash
# Clone repository
git clone https://github.com/your-username/bimasathi.git

# Install dependencies
npm install

# Configure environment variables
cp .env.example .env

# Run locally
npm run dev
```

*(Update this section according to your stack.)*

---

## 🔐 Security & Compliance Note

This project is built with consent-driven data collection principles.
Sensitive data is encrypted and role-restricted.

Future production deployments will align with Indian data protection guidelines.

---

## 🤝 Contributing

We welcome contributions focused on:

* Rural AI accessibility
* Regional language improvements
* Evidence intelligence models
* Fraud detection systems
* UI/UX for low-literacy users

---

## 🌱 Vision

To make crop insurance as simple, trusted, and instant as UPI — for every farmer in India.

---
