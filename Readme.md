# **IT Support Chatbot with AWS Lex, Lambda, API Gateway, and DynamoDB**

## **Project Overview**

This project implements an intelligent IT Support Chatbot designed to provide automated assistance for common IT troubleshooting queries. For questions beyond its knowledge base, it seamlessly escalates the query to a human agent via email and logs all interactions for future analysis and bot improvement.

The chatbot features a responsive web interface with a dark mode toggle for enhanced user experience.

## **Features**

* **Intelligent Chat Interface:** User-friendly and visually appealing web UI built with HTML, CSS (Tailwind CSS) and JavaScript.  
* **AWS Lex V2 Integration:** Leverages Amazon Lex for advanced Natural Language Understanding (NLU) to interpret user queries.  
* **Knowledge Base Powered:** Integrates with an AWS S3-backed Knowledge Base (e.g., PDFs of IT troubleshooting guides) allowing the bot to provide answers from provided documentation.  
* **Human Escalation:** Automatically identifies and escalates unresolvable user queries (via `FallbackIntent`) to a designated human support email using AWS Lambda and SES.  
* **Conversation Logging:** Stores all user queries and bot responses in an AWS DynamoDB table for analytics, auditing, and future bot training.  
* **Bot Confidence Tracking:** Logs Lex's confidence scores for each interpretation to the browser console, aiding in identifying areas for bot training improvement.  
* **Dynamic Response Formatting:** Client-side JavaScript enhances readability of multi-step responses by automatically formatting text (e.g., adding line breaks for numbered lists).  
* **Dark Mode Toggle:** Provides a toggle button for users to switch between light and dark themes, with their preference persisted across sessions using `localStorage`.  
* **Gratitude Handling:** A dedicated `ThankYouIntent` allows the bot to respond naturally to expressions of gratitude.  
* **Responsive Design:** Optimized for various screen sizes, ensuring a consistent experience on desktop and mobile devices.

## **Architecture**

The project leverages a serverless architecture on AWS for scalability and cost-effectiveness.

**Note:** The diagram below uses Mermaid syntax and will render as a visual graph directly on GitHub.

graph TD  
    subgraph Frontend (HTML/CSS/JS)  
        A\[User Browser\]  
    end

    subgraph AWS Cloud  
        subgraph Chatbot Backend  
            B(AWS Cognito Identity Pool)  
            C(AWS Lex V2 Bot)  
            D\[AWS S3 Bucket: Knowledge Base Docs\]  
        end

        subgraph Escalation & Logging  
            E(AWS API Gateway: /escalate)  
            F(AWS Lambda: EscalationFn)  
            G(AWS SES: Email Service)  
            H(AWS API Gateway: /store-query)  
            I(AWS Lambda: StoreQueryFn)  
            J\[AWS DynamoDB: ChatbotUserQueries Table\]  
            K\[AWS CloudWatch Logs\]  
        end  
    end

    A \-- Authenticates \--\> B  
    A \-- recognizeText \--\> C  
    A \-- POST /escalate \--\> E  
    A \-- POST /store-query \--\> H

    C \-- Queries \--\> D  
    C \-- Triggers \--\> F  
    E \-- Invokes (Lambda Proxy) \--\> F  
    F \-- Sends Email \--\> G  
    H \-- Invokes (Lambda Proxy) \--\> I  
    I \-- Writes Data \--\> J

    F \-- Logs \--\> K  
    I \-- Logs \--\> K  
    C \-- Logs \--\> K

## **Setup & Deployment Guide**

This guide covers setting up the necessary AWS services and deploying the frontend.

### **Prerequisites**

* An AWS Account  
* AWS CLI (configured) or AWS Management Console access  
* Node.js (for Lambda runtime)  
* Python (for local HTTP server)

### **1\. AWS DynamoDB Table Setup**

Create a table to store user queries and bot responses.

1. Go to **DynamoDB** in the AWS Console.  
2. Click **"Create table"**.  
3. **Table name:** `ChatbotUserQueries`  
4. **Partition key:** `sessionId` (String)  
5. **Sort key (Optional but Recommended):** `timestamp` (Number)  
6. Leave other settings as default and click **"Create table"**.

### **2\. AWS SES (Simple Email Service) Setup**

Set up SES to send escalation emails.

1. Go to **SES** in the AWS Console.  
2. In the region where your Lex bot and Lambda will reside (e.g., `us-east-1`), navigate to **"Verified identities"**.  
3. **Verify your sender email:** Click "Create identity," choose "Email address," enter `YOUR_SENDER_EMAIL_HERE` (your chosen sender email), and follow the verification steps (click link in the email).  
4. **Verify your recipient email (if in Sandbox Mode):** If your AWS account is still in SES Sandbox Mode, you *must* also verify `YOUR_RECIPIENT_EMAIL_HERE` (your chosen recipient email) here. For production, request moving out of Sandbox Mode.

### **3\. AWS Lambda Functions**

You'll need two Lambda functions: one for email escalation and one for DynamoDB storage.

#### **3.1. Lambda for Email Escalation (`escalateToHumanLambda`)**

1. Go to **Lambda** in the AWS Console.  
2. Click **"Create function"**.  
3. **Function name:** `escalateToHumanLambda`  
4. **Runtime:** `Node.js 18.x`  
5. **Execution role:** Create a new role with basic Lambda permissions.  
   * After creation, go to the Lambda's **"Configuration"** tab \> **"Permissions"**.  
   * Click the **"Role name"** to open IAM.

Attach a policy that grants `ses:SendEmail` and `ses:SendRawEmail` permissions (e.g., `AmazonSESFullAccess` for testing, or a custom policy limiting to specific resources).  
{  
    "Version": "2012-10-17",  
    "Statement": \[  
        {  
            "Effect": "Allow",  
            "Action": \[  
                "ses:SendEmail",  
                "ses:SendRawEmail"  
            \],  
            "Resource": "\*"  
        }  
    \]  
}

* 

Go to the **"Code"** tab and use the following:  
// index.mjs  
import { SESClient, SendEmailCommand } from "@aws-sdk/client-ses";

const sesClient \= new SESClient({ region: "us-east-1" }); // Ensure region matches your SES setup

export const handler \= async (event) \=\> {  
    console.log("Received event:", JSON.stringify(event, null, 2));

    const body \= JSON.parse(event.body);  
    const userQuery \= body.query;

    if (\!userQuery) {  
        return {  
            statusCode: 400,  
            headers: {  
                "Content-Type": "application/json",  
                "Access-Control-Allow-Origin": "\*"  
            },  
            body: JSON.stringify({ message: "Error: Query is missing." }),  
        };  
    }

    const SENDER\_EMAIL \= "YOUR\_SENDER\_EMAIL\_HERE"; // Your verified SES email  
    const RECIPIENT\_EMAIL \= "YOUR\_RECIPIENT\_EMAIL\_HERE"; // Your verified recipient email

    const params \= {  
        Source: SENDER\_EMAIL,  
        Destination: { ToAddresses: \[RECIPIENT\_EMAIL\] },  
        Message: {  
            Subject: { Data: "New Unresolved Query from IT Support Chatbot" },  
            Body: {  
                Html: { Data: \`\<p\>Hello Support Team,\</p\>\<p\>The IT Support chatbot could not resolve the following user query:\</p\>\<blockquote style="font-size: 1.1em; border-left: 3px solid \#ccc; padding-left: 15px; margin-left: 10px;"\>${userQuery}\</blockquote\>\<p\>Please review and consider adding this to the knowledge base or follow up with the user if their identity is known.\</p\>\` },  
                Text: { Data: \`A new query could not be resolved by the chatbot: "${userQuery}"\` }  
            },  
        },  
    };

    try {  
        await sesClient.send(new SendEmailCommand(params));  
        console.log("Email sent successfully.");  
        return {  
            statusCode: 200,  
            headers: {  
                "Content-Type": "application/json",  
                "Access-Control-Allow-Origin": "\*"  
            },  
            body: JSON.stringify({ message: "Query escalated successfully." }),  
        };  
    } catch (error) {  
        console.error("Failed to send email:", error);  
        return {  
            statusCode: 500,  
            headers: {  
                "Content-Type": "application/json",  
                "Access-Control-Allow-Origin": "\*"  
            },  
            body: JSON.stringify({ message: "Internal server error." }),  
        };  
    }  
};

6.   
7. Click **"Deploy"**.

#### **3.2. Lambda for Storing Queries (`storeUserQueryLambda`)**

1. Go to **Lambda** in the AWS Console.  
2. Click **"Create function"**.  
3. **Function name:** `storeUserQueryLambda`  
4. **Runtime:** `Node.js 18.x`  
5. **Execution role:** Create a new role with basic Lambda permissions.  
   * After creation, go to the Lambda's **"Configuration"** tab \> **"Permissions"**.  
   * Click the **"Role name"** to open IAM.

Attach a policy that grants `dynamodb:PutItem` permissions on your `ChatbotUserQueries` table.  
{  
    "Version": "2012-10-17",  
    "Statement": \[  
        {  
            "Effect": "Allow",  
            "Action": \[  
                "dynamodb:PutItem"  
            \],  
            "Resource": "arn:aws:dynamodb:us-east-1:YOUR\_AWS\_ACCOUNT\_ID:table/ChatbotUserQueries"  
        }  
    \]  
}

* 

  (Replace `YOUR_AWS_ACCOUNT_ID` with your actual AWS account ID.)

Go to the **"Code"** tab and use the following:  
// storeUserQueryLambda/index.mjs  
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";  
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const client \= new DynamoDBClient({ region: "us-east-1" }); // Ensure this matches your DynamoDB table's region  
const docClient \= DynamoDBDocumentClient.from(client);

export const handler \= async (event) \=\> {  
    console.log("Received event:", JSON.stringify(event, null, 2));

    let requestBody;  
    try {  
        // API Gateway Lambda Proxy integration passes body as a string  
        requestBody \= JSON.parse(event.body);  
    } catch (e) {  
        console.error("Error parsing request body:", e);  
        return {  
            statusCode: 400,  
            headers: {  
                "Content-Type": "application/json",  
                "Access-Control-Allow-Origin": "\*"  
            },  
            body: JSON.stringify({ message: "Invalid request body." }),  
        };  
    }

    const { sessionId, queryText, sender } \= requestBody;

    if (\!sessionId || \!queryText || \!sender) {  
        console.error("Missing required fields: sessionId, queryText, or sender.");  
        return {  
            statusCode: 400,  
            headers: {  
                "Content-Type": "application/json",  
                "Access-Control-Allow-Origin": "\*"  
            },  
            body: JSON.stringify({ message: "Missing required fields (sessionId, queryText, sender)." }),  
        };  
    }

    const timestamp \= Date.now(); // Milliseconds since epoch

    const params \= {  
        TableName: "ChatbotUserQueries", // Your DynamoDB table name  
        Item: {  
            sessionId: sessionId,  
            timestamp: timestamp,  
            queryText: queryText,  
            sender: sender, // 'user' or 'bot'  
        },  
    };

    try {  
        await docClient.send(new PutCommand(params));  
        console.log("Query stored successfully in DynamoDB.");  
        return {  
            statusCode: 200,  
            headers: {  
                "Content-Type": "application/json",  
                "Access-Control-Allow-Origin": "\*"  
            },  
            body: JSON.stringify({ message: "Query stored successfully." }),  
        };  
    } catch (error) {  
        console.error("Failed to store query in DynamoDB:", error);  
        return {  
            statusCode: 500,  
            headers: {  
                "Content-Type": "application/json",  
                "Access-Control-Allow-Origin": "\*"  
            },  
            body: JSON.stringify({ message: "Internal server error: Could not store query." }),  
        };  
    }  
};

6.   
7. Click **"Deploy"**.

### **4\. AWS API Gateway Setup**

You'll have two API Gateway endpoints. One already exists for escalation, and you'll add a new one for storage.

#### **4.1. Existing Escalation Endpoint (`/escalate`)**

Ensure your existing API Gateway for escalation (`YOUR_ESCALATE_API_URL_HERE`) has CORS configured correctly for your frontend's origin (`http://localhost:8000` or your deployed domain).

#### **4.2. New Storage Endpoint (`/store-query`)**

1. Go to **API Gateway** in the AWS Console. You can use an existing REST API or create a new one (e.g., `ChatbotQueryStorageAPI`).  
2. If creating new: **"Create API"** \> **"REST API"** \> **"Build"** \> **"New API"**. Give it a name like `ChatbotQueryStorageAPI`.  
3. **Create Resource:**  
   * Under "Resources," click **"Actions"** \> **"Create Resource"**.  
   * **Uncheck "Proxy resource"**.  
   * **Resource Name:** `store-query`  
   * **Resource Path:** `/store-query`  
   * Check **"CORS (Cross Origin Resource Sharing)"**.  
   * Click **"Create Resource"**.  
4. **Create Method:**  
   * With `/store-query` selected, click **"Actions"** \> **"Create Method"**.  
   * Select **`POST`** and click the checkmark.  
   * **Integration type:** `Lambda Function`.  
   * **CHECK the box for "Use Lambda Proxy integration"**. (This is critical\!)  
   * **Lambda Region:** `us-east-1`.  
   * **Lambda Function:** Select `storeUserQueryLambda`.  
   * Click **"Save"**. Agree to add permissions.  
5. **Deploy API:**  
   * Click **"Actions"** \> **"Deploy API"**.  
   * Choose an existing stage or `[New Stage]` (e.g., `dev`).  
   * Click **"Deploy"**.  
   * **Copy the "Invoke URL"**. This is your `API_GATEWAY_STORAGE_URL`.

### **5\. AWS Cognito Identity Pool Setup**

This allows your frontend to access Lex and API Gateway without explicit user login.

1. Go to **Cognito** in the AWS Console.  
2. Click **"Manage Identity Pools"** (if not already there).  
3. Click **"Create new identity pool"**.  
4. **Identity pool name:** `ChatbotIdentityPool`  
5. Check **"Enable access to unauthenticated identities"**.  
6. Click **"Create Pool"**.  
7. On the next screen ("Your Cognito Identity Pool has been successfully created"), expand **"View Details"**.  
8. Click **"Allow"** to create default IAM roles.  
9. Go to the IAM Console to verify the created roles (e.g., `Cognito_ChatbotIdentityPoolUnauth_Role`). This role should have permissions for `lexv2-runtime:RecognizeText`, `execute-api:Invoke` on your API Gateway endpoints, etc. If not, add them.

### **6\. AWS Lex V2 Bot Setup**

Your existing Lex bot, ensure the following:

1. Go to **Amazon Lex V2** Console.  
2. Select your bot (`YOUR_LEX_BOT_ID_HERE`).  
3. **`FallbackIntent` Configuration:**  
   * Switch to the **`Draft` version** of your bot.  
   * Navigate to **"Intents"** \> `FallbackIntent`.  
   * Under **"Fulfillment"**, ensure **"Run a Lambda function to fulfill the intent..."** is enabled.  
   * Select `escalateToHumanLambda`.  
   * Ensure you have "On successful fulfillment," "In case of failure," and "Timeout" messages configured.  
   * **Save Intent**.  
4. **Knowledge Base Integration:** Ensure your S3-backed Knowledge Base is correctly associated with your bot's `Draft` version and that data ingestion is "Ready."  
5. **Build Bot:** After any changes in `Draft`, click **"Build"** (top right) and wait for completion.  
6. **Bot Alias:** Go to **"Bot aliases"**.  
   * Click on `YOUR_LEX_BOT_ALIAS_ID_HERE`.  
   * Ensure **"Bot version"** is set to **"Draft"** (for development) or a specific version you've published.

### **7\. Frontend Deployment (HTML/JavaScript)**

The `index.html` file on your local machine.

1. **Open your `index.html` file.**  
2. **Update Placeholders:**  
   * `IdentityPoolId`: `YOUR_COGNITO_IDENTITY_POOL_ID_HERE` (from your Cognito Identity Pool).  
   * `LEX_BOT_ID`: `YOUR_LEX_BOT_ID_HERE` (from your Lex bot).  
   * `LEX_BOT_ALIAS_ID`: `YOUR_LEX_BOT_ALIAS_ID_HERE` (from your Lex bot alias).  
   * `API_GATEWAY_ESCALATE_URL`: `YOUR_ESCALATE_API_URL_HERE` (your existing escalation API URL).  
   * `API_GATEWAY_STORAGE_URL`: **PASTE THE INVOKE URL of your `/store-query` API Gateway here.**  
3. **Save the `index.html` file.**

**Serve Locally:** Open your terminal, navigate to the directory containing `index.html`, and run:  
python \-m http.server 8000

4.   
5. **Open in Browser:** Access your chatbot at `http://localhost:8000/index.html`.

## **Troubleshooting Guide**

This section summarizes common issues and their solutions.

### **1\. `Access to fetch at '...' from origin 'null' has been blocked by CORS policy`**

* **Problem:** Browser blocks requests when `index.html` is opened directly (`file:///` URL).  
* **Solution:** Always serve your `index.html` via a local HTTP server (e.g., `python -m http.server 8000`).  
* **API Gateway CORS:** Ensure API Gateway methods (especially `POST` for `/escalate` and `/store-query`) have CORS enabled and allow your frontend's origin (e.g., `http://localhost:8000`). Redeploy API after changes.

### **2\. `500 Internal Server Error` from API Gateway**

* **Problem:** API Gateway endpoint successfully receives request but Lambda function fails.  
* **Solution:**  
  1. **Check Lambda CloudWatch Logs:** Go to the Lambda function in console \-\> "Monitor" tab \-\> "View CloudWatch logs." Look for `ERROR` messages and stack traces (e.g., `MessageRejected` from SES, or DynamoDB permission errors).  
  2. **Lambda IAM Permissions:** Ensure the Lambda's execution role has the necessary permissions (e.g., `ses:SendEmail` for escalation, `dynamodb:PutItem` for storage).

### **3\. `SyntaxError: Unexpected token u in JSON at position 0` in Lambda logs**

* **Problem:** Lambda function receives an `event.body` that isn't a valid JSON string (often `undefined` or `null`).  
* **Solution:** This typically means **API Gateway Lambda Proxy Integration is not enabled**.  
  1. In API Gateway, navigate to the method (`POST /store-query`).  
  2. In "Integration Request," ensure the **"Use Lambda Proxy integration"** checkbox is **checked**.  
  3. Redeploy the API after making this change.

### **4\. Chat UI elements (Dark Mode Toggle, Send Button) missing or not working / `ReferenceError: ... is not defined`**

* **Problem:** HTML elements or JavaScript variables for them are not found.  
* **Solution:**  
  1. **Verify HTML Structure:** Double-check that the `<button id="themeToggle">` and the send button's `<svg>` are correctly present in your `index.html` as provided in the instructions.  
  2. **JavaScript Initialization Order:** Ensure all `document.getElementById()` calls and subsequent AWS SDK initialization (`AWS.config`, `lexruntime`) are wrapped within `window.addEventListener('DOMContentLoaded', () => { ... });`. This guarantees elements exist before JavaScript tries to access them.

### **5\. Bot Always Escalates / Knowledge Base Not Working in Frontend**

* **Problem:** Lex bot responds with fallback message and escalates, even for queries that should be answered by the Knowledge Base (works in Lex console test).  
* **Solution:**  
  1. **Frontend Logic:** Adjust your `sendMessage` function's Lex response handling. Prioritize checking `if (data.messages && data.messages.length > 0)` to display *any* messages Lex returns (including KB answers). Only if `data.messages` is empty should you proceed with the escalation logic.  
  2. **Lex Knowledge Base Status:** In Lex console, verify your Knowledge Base status is "Ready" and there are no ingestion errors in its data sources.  
  3. **Lex Bot Alias Version:** Ensure your bot alias (`YOUR_LEX_BOT_ALIAS_ID_HERE`) is pointing to the `Draft` version (for development) or a specific published version that includes the Knowledge Base integration.

### **6\. New Lex Intent (e.g., `ThankYouIntent`) Not Working / Cannot Create Intent in Version X**

* **Problem:** Newly created Lex intents are not recognized, or you cannot modify a specific Lex bot version.  
* **Solution:**  
  1. **Lex Version Immutability:** Lex bot versions are read-only. **All development (creating/modifying intents, responses, Lambda integrations) must be done in the `Draft` version of your bot.**  
  2. **Save and Build:** After making changes in `Draft`, always click **"Save intent"** and then **"Build"** the bot (top-right corner).  
  3. **Update Bot Alias:** For your frontend to use the latest changes, update your Lex bot alias (`YOUR_LEX_BOT_ALIAS_ID_HERE`) to point to the `Draft` version (or a newly published version if going to production).

## **Usage**

1. Open the `index.html` file in your browser (via `http://localhost:8000`).  
2. Type your IT support query in the input box and press Enter or click the send button.  
3. The bot will attempt to answer using its Knowledge Base.  
4. If the query cannot be resolved, it will display a fallback message and automatically send an email to the configured human support address.  
5. All interactions are logged in your DynamoDB table.  
6. Use the toggle button in the header to switch between light and dark modes.

## **Future Enhancements**

* **Admin Dashboard:** Create a separate web interface to view the stored user queries in DynamoDB for analytics and manual review.  
* **Contextual Conversations:** Implement more complex multi-turn dialogues in Lex.  
* **Sentiment Analysis:** Use Amazon Comprehend to analyze user sentiment for better escalation or response tailoring.  
* **Voice Input/Output:** Integrate speech-to-text and text-to-speech capabilities.  
* **User Feedback:** Add a "Was this helpful?" rating system for bot responses to gather explicit feedback.  
* **Advanced Formatting:** Implement richer formatting for bot responses (e.g., Markdown rendering for lists, bolding) if Lex provides content in such formats.  
* **Monitoring & Alerts:** Set up CloudWatch alarms for Lambda errors or high `FallbackIntent` rates.

## 

## **Contributing**

Feel free to fork this repository, open issues, or submit pull requests.

## **License**

This project is open-source and available under the MIT License.

