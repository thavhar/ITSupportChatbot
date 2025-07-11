

# 🚀 **IT Support Chatbot with AWS Lex, Lambda, API Gateway, and DynamoDB**

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![AWS](https://img.shields.io/badge/built%20with-AWS-orange.svg)](https://aws.amazon.com/)

---

## 📚 **Table of Contents**

* [Project Overview](#project-overview)
* [Features](#features)
* [Architecture](#architecture)
* [Setup & Deployment Guide](#setup--deployment-guide)

  * [Prerequisites](#prerequisites)
  * [1. DynamoDB Setup](#1-aws-dynamodb-table-setup)
  * [2. SES Setup](#2-aws-ses-simple-email-service-setup)
  * [3. Lambda Functions](#3-aws-lambda-functions)
  * [4. API Gateway](#4-aws-api-gateway-setup)
  * [5. Cognito Setup](#5-aws-cognito-identity-pool-setup)
  * [6. Lex V2 Bot Setup](#6-aws-lex-v2-bot-setup)
  * [7. Frontend Deployment](#7-frontend-deployment-htmljavascript)
* [Troubleshooting Guide](#troubleshooting-guide)
* [Usage](#usage)
* [Future Enhancements](#future-enhancements)
* [Contributing](#contributing)
* [License](#license)

---

## 🎯 **Project Overview**

This project implements an intelligent IT Support Chatbot designed to provide automated assistance for common IT troubleshooting queries.

For questions beyond its knowledge base, it seamlessly escalates the query to a human agent via email and logs all interactions for future analysis and bot improvement.

✨ **Bonus:** The chatbot features a responsive web interface with a dark mode toggle for enhanced user experience.

---

## ✨ **Features**

✅ **Intelligent Chat Interface**

* User-friendly web UI built with HTML, Tailwind CSS, and JavaScript
* Responsive design for desktop and mobile

✅ **AWS Lex V2 Integration**

* Advanced natural language understanding (NLU)
* Recognizes user intents and manages conversations

✅ **Knowledge Base**

* Integrated with an AWS S3-backed Knowledge Base (e.g. PDFs of IT guides)
* Answers pulled directly from provided documentation

✅ **Human Escalation**

* Automatically escalates unresolved queries via Lex’s `FallbackIntent`
* Sends email to support team through AWS Lambda + SES

✅ **Conversation Logging**

* Logs all user queries and bot responses in DynamoDB
* Useful for analytics, audits, and bot improvement

✅ **Bot Confidence Tracking**

* Logs Lex’s confidence scores to browser console for analysis

✅ **Dynamic Response Formatting**

* Client-side JavaScript formats multi-step responses (e.g. numbered lists) for better readability

✅ **Dark Mode Toggle**

* Users can switch themes
* Preferences saved in `localStorage`

✅ **Gratitude Handling**

* `ThankYouIntent` enables the bot to respond naturally to user gratitude

---

## 🗺️ **Architecture**

This chatbot uses a serverless architecture for scalability and cost efficiency.


graph TD
    subgraph Frontend (HTML/CSS/JS)
        A[User Browser]
    end

    subgraph AWS Cloud
        subgraph Chatbot Backend
            B(AWS Cognito Identity Pool)
            C(AWS Lex V2 Bot)
            D[AWS S3: Knowledge Base Docs]
        end

        subgraph Escalation & Logging
            E(API Gateway: /escalate)
            F(Lambda: EscalationFn)
            G(SES: Email Service)
            H(API Gateway: /store-query)
            I(Lambda: StoreQueryFn)
            J[DynamoDB: ChatbotUserQueries Table]
            K[CloudWatch Logs]
        end
    end

    A -->|Authenticates| B
    A -->|recognizeText| C
    A -->|POST /escalate| E
    A -->|POST /store-query| H
    C -->|Queries| D
    C -->|Triggers| F
    E -->|Lambda Proxy| F
    F -->|Sends Email| G
    H -->|Lambda Proxy| I
    I -->|Writes Data| J
    F -->|Logs| K
    I -->|Logs| K
    C -->|Logs| K
```

---

## ⚙️ **Setup & Deployment Guide**

### Prerequisites

* AWS Account
* AWS CLI (configured) OR AWS Console Access
* Node.js (for Lambda runtime)
* Python (for local HTTP server)

---

### **1. AWS DynamoDB Table Setup**

1. Go to DynamoDB → **Create table**
2. Table name: `ChatbotUserQueries`
3. Partition key: `sessionId` (String)
4. Sort key: `timestamp` (Number, optional but recommended)
5. Save table

---

### **2. AWS SES (Simple Email Service) Setup**

1. Go to SES → Verified identities
2. Verify sender email (and recipient email if in sandbox mode)
3. Move out of sandbox for production if needed

---

### **3. AWS Lambda Functions**

#### **3.1 Escalation Function**

* Name: `escalateToHumanLambda`
* Runtime: Node.js 18.x
* Permissions:

  * `ses:SendEmail`
  * `ses:SendRawEmail`

**Code:** (Use your original snippet here for brevity)

#### **3.2 Store Query Function**

* Name: `storeUserQueryLambda`
* Runtime: Node.js 18.x
* Permissions:

  * `dynamodb:PutItem` on `ChatbotUserQueries`

**Code:** (Use your original snippet here)

---

### **4. AWS API Gateway Setup**

Two endpoints:

* `/escalate` → triggers `escalateToHumanLambda`
* `/store-query` → triggers `storeUserQueryLambda`

Ensure:

* Lambda Proxy integration enabled
* CORS configured for your frontend origin

---

### **5. AWS Cognito Identity Pool Setup**

* Create identity pool
* Enable unauthenticated identities
* Attach permissions:

  * `lexv2-runtime:RecognizeText`
  * `execute-api:Invoke` on API Gateway endpoints

---

### **6. AWS Lex V2 Bot Setup**

* Create Lex bot
* Add intents including:

  * FallbackIntent
  * ThankYouIntent
* Integrate S3 knowledge base
* Point bot alias to Draft or desired version

---

### **7. Frontend Deployment (HTML/JS)**

* Update placeholders in `index.html`:

  * Cognito IdentityPoolId
  * Lex Bot ID and Alias
  * API URLs for `/escalate` and `/store-query`
* Serve locally:

  ```bash
  python -m http.server 8000
  ```
* Open browser at:

  ```
  http://localhost:8000
  ```

---

## 🛠️ **Troubleshooting Guide**

| **Issue**                       | **Possible Cause & Fix**                                             |
| ------------------------------- | -------------------------------------------------------------------- |
| **CORS Error**                  | Serve frontend locally. Enable CORS in API Gateway.                  |
| **500 Error**                   | Check Lambda logs in CloudWatch for errors.                          |
| **Unexpected Token Error**      | Ensure Lambda Proxy integration is enabled in API Gateway.           |
| **UI Elements Missing**         | Check HTML IDs and ensure JS runs after DOM load.                    |
| **Lex always escalates**        | Check knowledge base status and Lex alias version.                   |
| **New Lex intents not working** | Always build the bot after changes and point alias to Draft version. |

---

## 🖥️ **Usage**

1. Run local server
2. Open chatbot page
3. Ask IT support questions
4. Chatbot attempts to answer or escalates via email
5. All chats stored in DynamoDB
6. Toggle dark mode as desired!

---

## 🚀 **Future Enhancements**

* Admin dashboard for DynamoDB logs
* Multi-turn dialogues
* Sentiment analysis (Amazon Comprehend)
* Voice integration (speech-to-text/text-to-speech)
* Feedback ratings for responses
* Richer response formatting
* CloudWatch alerts for high fallback rates

---

## 🤝 **Contributing**

Feel free to fork this repository, open issues, or submit pull requests!

---

## 📄 **License**

This project is open-source under the [MIT License](LICENSE).

---

**🔗 Pro tip:** Add your own screenshots or GIFs of the chatbot in action to the top of the README for extra polish!

Would you like me to embed your actual Lambda code snippets directly into this README as well? Or leave them as separate files? Let me know how you’d prefer it!
