
-----

# **IT Support Chatbot: End-to-End Setup Guide**

-----

## **OVERVIEW**

This guide provides all the necessary steps to set up, configure, and deploy the complete IT support chatbot application. It is divided into three main parts:

1.  **Backend Setup**: Creating and configuring all required AWS services (SES, Lambda, API Gateway, Cognito, Lex V2).
2.  **Frontend Code & Configuration**: Preparing and configuring the `index.html` file.
3.  **Testing & Deployment**: Running the chatbot locally and deploying it to the web.

-----

## **PART 1: BACKEND INFRASTRUCTURE SETUP (AWS)**

Before configuring the frontend, you must set up the cloud infrastructure.

### **Step 1.1: Configure Amazon SES (Simple Email Service)**

SES is used for sending escalation emails to the support team.

1.  Navigate to the **Amazon SES** console.
2.  Go to **Verified identities**.
3.  Click **Create identity**.
4.  Verify two email addresses:
      * The email you want to send notifications **FROM** (e.g., `bot-notifications@your-domain.com`).
      * The email of the support team that will receive the notifications **TO** (e.g., `support-team@your-domain.com`).
      * Complete the verification process by clicking the links sent to these email inboxes.

### **Step 1.2: Create the Escalation Lambda Function**

This function handles the email sending process upon fallback.

1.  Navigate to the **AWS Lambda** console.
2.  Click **Create function**.
      * **Name**: `ITSupportBotFallbackHandler`
      * **Runtime**: `Node.js` (e.g., 18.x or newer)
      * **Permissions**: Choose "Create a new role with basic Lambda permissions".
3.  Once created, navigate to the **Configuration** \> **Permissions** tab. Click on the **Role name** to open the IAM console.
4.  In the IAM console, click **Add permissions** \> **Attach policies**. Search for and attach the `AmazonSESFullAccess` policy.
5.  Go back to the Lambda function's **Code** tab. Replace the content of `index.mjs` with the following code:

<!-- end list -->

```javascript
// index.mjs for Lambda
import { SESClient, SendEmailCommand } from "@aws-sdk/client-ses";

const sesClient = new SESClient({ region: "us-east-1" }); // Make sure this region matches your SES identity

export const handler = async (event) => {
    // --- CONFIGURE YOUR EMAIL DETAILS ---
    const SENDER_EMAIL = "your-verified-sender-email@example.com";     // <-- IMPORTANT: Replace with your FROM email
    const RECIPIENT_EMAIL = "your-support-team-email@example.com";    // <-- IMPORTANT: Replace with your TO email

    let userQuery;
    try {
        const body = JSON.parse(event.body);
        userQuery = body.query;
    } catch (e) {
        console.error("Failed to parse event body:", e);
        return { statusCode: 400, body: JSON.stringify({ message: "Invalid request body." }) };
    }

    if (!userQuery) {
        return { statusCode: 400, body: JSON.stringify({ message: "Query is missing." }) };
    }

    const params = {
        Source: SENDER_EMAIL,
        Destination: { ToAddresses: [RECIPIENT_EMAIL] },
        Message: {
            Subject: { Data: "New Unresolved Query from IT Support Chatbot" },
            Body: {
                Html: { Data: `<p>The IT Support chatbot could not resolve the following user query:</p><blockquote>${userQuery}</blockquote>` },
                Text: { Data: `A new query could not be resolved by the chatbot: "${userQuery}"` }
            },
        },
    };

    try {
        await sesClient.send(new SendEmailCommand(params));
        return {
            statusCode: 200,
            headers: { "Access-Control-Allow-Origin": "*" }, // Allow CORS
            body: JSON.stringify({ message: "Query escalated successfully." }),
        };
    } catch (error) {
        console.error("Failed to send email:", error);
        return {
            statusCode: 500,
            headers: { "Access-Control-Allow-Origin": "*" }, // Allow CORS
            body: JSON.stringify({ message: "Internal server error." }),
        };
    }
};
```

6.  **Replace the placeholder emails** in the code above. Click **Deploy** to save your changes.

### **Step 1.3: Create the API Gateway Endpoint**

This creates a public URL to trigger the Lambda function from the frontend.

1.  Navigate to the **Amazon API Gateway** console.
2.  Click **Create API** and choose **HTTP API** \> **Build**.
3.  Name the API (e.g., `ChatbotFallbackAPI`).
4.  Configure a single integration:
      * **Integration type**: `Lambda`
      * **Lambda function**: `ITSupportBotFallbackHandler`
5.  Configure a single route:
      * **Method**: `POST`
      * **Resource path**: `/escalate`
      * **Integration target**: Should be set to your Lambda function.
6.  Create the API. Once done, find and copy the **Invoke URL**.
7.  In your new API's settings, go to the **CORS** section. Configure it with the following:
      * **Access-Control-Allow-Origin**: `*` (or your specific frontend domain for better security).
      * **Access-Control-Allow-Methods**: `POST`
      * **Access-Control-Allow-Headers**: `Content-Type`
8.  Click **Save**.

### **Step 1.4: Set Up Amazon Cognito for Secure Frontend Credentials**

Cognito provides temporary credentials for the frontend to interact with Lex.

1.  Navigate to the **Amazon Cognito** console.
2.  Click **Create identity pool**.
3.  Name it (e.g., `ChatbotIdentityPool`).
4.  Check the box for **Enable access to unauthenticated identities**.
5.  Click **Create Pool**. Click **Allow** to create new IAM roles.
6.  Find and copy the **Identity pool ID**.
7.  Go to the **IAM** console \> **Roles**. Find the unauthenticated role you just created (it will have "unauth" in the name). Attach a new inline policy that allows the `lex:RecognizeText` action.

### **Step 1.5: Get Your Amazon Lex V2 Bot Details**

1.  Navigate to your bot in the **Amazon Lex V2** console.
2.  Ensure the built-in **FallbackIntent** is enabled (this will be used for human escalation).
3.  Note the following details from your bot's main page and deployment alias:
      * **Bot ID**
      * **Bot Alias ID**
      * **Locale ID** (e.g., `en_US`)

-----

## **PART 2: FRONTEND CODE & CONFIGURATION**

Now, you will configure the `index.html` file that serves the chatbot interface.

### **Step 2.1: The Complete `index.html` Code**

Create a file named `index.html` and paste the entire block of code below into it.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AWS Lex Chatbot Interface</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        html, body { height: 100%; margin: 0; padding: 0; }
        body { font-family: 'Inter', sans-serif; background: linear-gradient(135deg, #FF6B6B, #4ECDC4); display: flex; align-items: center; justify-content: center; min-height: 100vh; padding: 1rem; box-sizing: border-box; transition: background 0.5s ease; }
        body.dark-mode { background: linear-gradient(135deg, #2C3E50, #000000); }
        .chat-container { background-color: #ffffff; border-radius: 1.5rem; box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.4); display: flex; flex-direction: column; width: 100%; max-width: 28rem; height: 85vh; overflow: hidden; transition: transform 0.3s ease-out, background-color 0.5s ease, box-shadow 0.5s ease; border: none; }
        .chat-container.dark-mode { background-color: #1a202c; box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.7); }
        .chat-header { background: linear-gradient(to right, #00B4D8, #4895EF); color: #ffffff; padding: 1.5rem; text-align: center; font-size: 1.75rem; font-weight: 800; border-top-left-radius: 1.5rem; border-top-right-radius: 1.5rem; box-shadow: 0 15px 20px -5px rgba(0, 0, 0, 0.2); border-bottom: 3px solid rgba(255,255,255,0.4); text-shadow: 2px 2px 4px rgba(0,0,0,0.2); display: flex; justify-content: space-between; align-items: center; position: relative; transition: background 0.5s ease; }
        .chat-header.dark-mode { background: linear-gradient(to right, #2C3E50, #4A5568); color: #E2E8F0; text-shadow: 2px 2px 4px rgba(0,0,0,0.4); }
        .chat-header-title { flex-grow: 1; text-align: center; margin-right: 2.5rem; }
        .chat-box { flex-grow: 1; padding: 1rem; overflow-y: auto; background-color: #E0E7FF; scrollbar-width: thin; scrollbar-color: #7B68EE #E0E7FF; transition: background-color 0.5s ease; }
        .chat-box.dark-mode { background-color: #2D3748; scrollbar-color: #63B3ED #2D3748; }
        .user-message, .bot-message { padding: 0.75rem; border-radius: 0.75rem; max-width: 80%; margin-top: 0.5rem; margin-bottom: 0.5rem; box-shadow: 0 6px 10px -2px rgba(0, 0, 0, 0.15); animation: fadeIn 0.3s ease-out forwards; transition: background-color 0.5s ease, color 0.5s ease, box-shadow 0.5s ease; }
        .user-message { background-color: #5D2E8E; color: #ffffff; margin-left: auto; margin-right: 0; border-bottom-right-radius: 0.25rem; }
        .user-message.dark-mode { background-color: #4A5568; color: #E2E8F0; }
        .bot-message { background-color: #A2D9FF; color: #2F3640; margin-left: 0; margin-right: auto; border-bottom-left-radius: 0.25rem; }
        .bot-message.dark-mode { background-color: #63B3ED; color: #1A202C; }
        .chat-input { display: flex; padding: 1rem; border-top: 1px solid #e2e8f0; background-color: #ffffff; transition: border-top 0.5s ease, background-color 0.5s ease; }
        .chat-input.dark-mode { border-top: 1px solid #4A5568; background-color: #1A202C; }
        .chat-input input { flex-grow: 1; padding: 0.75rem; border: 1px solid #cbd5e0; border-radius: 9999px; outline: none; box-shadow: 0 2px 4px 0 rgba(0, 0, 0, 0.1); color: #4a5568; transition: border-color 0.2s, box-shadow 0.2s, background-color 0.5s ease, color 0.5s ease; }
        .chat-input input:focus { border-color: #00B4D8; box-shadow: 0 0 0 4px rgba(0, 180, 216, 0.6); }
        .chat-input input.dark-mode { background-color: #2D3748; border-color: #4A5568; color: #E2E8F0; }
        .chat-input button { margin-left: 0.75rem; background: linear-gradient(to right, #FF6B6B, #FFCD38); color: #ffffff; padding: 0.75rem; border-radius: 9999px; font-weight: 700; box-shadow: 0 12px 20px -5px rgba(0, 0, 0, 0.3); transition: all 0.3s ease-in-out; cursor: pointer; }
        .chat-input button:hover { transform: scale(1.1); }
        .theme-toggle { position: absolute; right: 1.5rem; top: 50%; transform: translateY(-50%); background: none; border: none; cursor: pointer; font-size: 1.5rem; color: #ffffff; transition: color 0.3s ease; }
        .theme-toggle:hover { color: #FFD700; }
        .theme-toggle svg.sun { display: block; }
        .theme-toggle svg.moon { display: none; }
        .dark-mode .theme-toggle svg.sun { display: none; }
        .dark-mode .theme-toggle svg.moon { display: block; color: #FFD700; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(15px); } to { opacity: 1; transform: translateY(0); } }
    </style>
</head>
<body>
    <div class="chat-container" id="chatApp">
        <div class="chat-header" id="chatHeader">
            <div class="chat-header-title">IT SUPPORT CHATBOT</div>
            <button class="theme-toggle" id="themeToggle" aria-label="Toggle dark mode">
                <svg class="w-6 h-6 sun" fill="currentColor" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><path d="M10 2a1 1 0 011 1v1a1 1 0 11-2 0V3a1 1 0 011-1zm4 8a4 4 0 11-8 0 4 4 0 018 0zm-.464 4.95l.707.707a1 1 0 001.414-1.414l-.707-.707a1 1 0 00-1.414 1.414zm2.12-10.607a1 1 0 010 1.414l-.706.707a1 1 0 11-1.414-1.414l.707-.707a1 1 0 011.414 0zM17 11a1 1 0 100-2h-1a1 1 0 100 2h1zm-13 0a1 1 0 100-2H2a1 1 0 100 2h1zm-.464-4.95l-.707-.707a1 1 0 00-1.414 1.414l.707.707a1 1 0 001.414-1.414zm2.12 10.607a1 1 0 010-1.414l-.706-.707a1 1 0 111.414-1.414l.707.707a1 1 0 01-1.414 1.414zM10 15a1 1 0 011 1v1a1 1 0 11-2 0v-1a1 1 0 011-1zM10 4a6 6 0 100 12 6 6 0 000-12z" clip-rule="evenodd" fill-rule="evenodd"></path></svg>
                <svg class="w-6 h-6 moon" fill="currentColor" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><path d="M17.293 13.293A8 8 0 016.707 2.707a8.001 8.001 0 1010.586 10.586z"></path></svg>
            </button>
        </div>
        <div class="chat-box" id="chatBox"></div>
        <div class="chat-input" id="chatInput">
            <input type="text" id="userInput" placeholder="Type your message..." aria-label="Chat input field">
            <button onclick="sendMessage()" aria-label="Send message">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M14 5l7 7m0 0l-7 7m7-7H3" />
                </svg>
            </button>
        </div>
    </div>

    <script src="https://sdk.amazonaws.com/js/aws-sdk-2.1158.0.min.js"></script>

    <script>
        // --- AWS CONFIGURATION --- (NON-DOM-DEPENDENT)
        AWS.config.region = 'us-east-1';
        AWS.config.credentials = new AWS.CognitoIdentityCredentials({
            IdentityPoolId: 'YOUR_COGNITO_IDENTITY_POOL_ID' // <-- PASTE YOUR ID HERE
        });
        const lexruntime = new AWS.LexRuntimeV2();
        const LEX_BOT_ID = 'YOUR_LEX_BOT_ID';                 // <-- PASTE YOUR ID HERE
        const LEX_BOT_ALIAS_ID = 'YOUR_LEX_BOT_ALIAS_ID';     // <-- PASTE YOUR ID HERE
        const LEX_LOCALE_ID = 'en_US';                        // <-- PASTE YOUR LOCALE HERE
        const API_GATEWAY_INVOKE_URL = 'YOUR_API_GATEWAY_INVOKE_URL'; // <-- PASTE YOUR URL HERE
        const sessionId = 'web-user-' + Date.now();

        // --- FUNCTION DEFINITIONS ---
        function displayMessage(message, sender) {
            const chatBox = document.getElementById('chatBox');
            const msgDiv = document.createElement('div');
            msgDiv.className = sender === 'user' ? 'user-message' : 'bot-message';
            msgDiv.textContent = message;
            if (document.body.classList.contains('dark-mode')) {
                msgDiv.classList.add('dark-mode');
            }
            chatBox.appendChild(msgDiv);
            chatBox.scrollTop = chatBox.scrollHeight;
        }

        async function escalateToHuman(queryText) {
            try {
                const response = await fetch(API_GATEWAY_INVOKE_URL + '/escalate', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ query: queryText })
                });
                if (!response.ok) throw new Error(`API Gateway request failed`);
                console.log("Query successfully escalated.");
            } catch (error) {
                console.error('Error escalating query:', error);
                displayMessage('Sorry, there was an issue escalating your request.', 'bot');
            }
        }

        function sendMessage() {
            const userInput = document.getElementById('userInput');
            const messageText = userInput.value.trim();
            if (messageText === '') return;

            displayMessage(messageText, 'user');
            userInput.value = '';

            const params = {
                botId: LEX_BOT_ID,
                botAliasId: LEX_BOT_ALIAS_ID,
                localeId: LEX_LOCALE_ID,
                sessionId: sessionId,
                text: messageText,
            };

            lexruntime.recognizeText(params, function(err, data) {
                if (err) {
                    console.error('Error with Lex:', err);
                    displayMessage('Error: Could not connect to the chatbot.', 'bot');
                } else {
                    if (data.messages && data.messages.length > 0) {
                        data.messages.forEach(message => {
                            if (message.contentType === 'PlainText') {
                                displayMessage(message.content, 'bot');
                            }
                        });
                    } else {
                        displayMessage('I am sorry, I could not find an answer. Your query has been forwarded to a human agent.', 'bot');
                        escalateToHuman(messageText);
                    }
                }
            });
        }

        function applyDarkMode(isDarkMode) {
            const elements = [
                document.body,
                document.getElementById('chatApp'),
                document.getElementById('chatHeader'),
                document.getElementById('chatBox'),
                document.getElementById('chatInput'),
                document.getElementById('userInput'),
                document.querySelector('.chat-input button')
            ];
            elements.forEach(el => el.classList.toggle('dark-mode', isDarkMode));
            localStorage.setItem('darkModeEnabled', isDarkMode);
        }

        // --- SCRIPT EXECUTION STARTS HERE ---
        window.addEventListener('DOMContentLoaded', () => {
            const userInput = document.getElementById('userInput');
            const themeToggle = document.getElementById('themeToggle');

            // Check for saved dark mode preference
            if (localStorage.getItem('darkModeEnabled') === 'true') {
                applyDarkMode(true);
            }

            // Attach Event Listeners
            themeToggle.addEventListener('click', () => {
                applyDarkMode(!document.body.classList.contains('dark-mode'));
            });
            userInput.addEventListener('keypress', (event) => {
                if (event.key === 'Enter') {
                    event.preventDefault();
                    sendMessage();
                }
            });

            // Initial welcome message
            displayMessage('Hello! How can I assist you today?', 'bot');
        });
    </script>
</body>
</html>
```

### **Step 2.2: Update Configuration Placeholders**

Open the `index.html` file you just created. Locate the `// --- AWS CONFIGURATION ---` section within the `<script>` tag and replace the placeholders with the AWS details you collected in Part 1:

  * **`YOUR_COGNITO_IDENTITY_POOL_ID`**: Replace with your Cognito Identity Pool ID.
  * **`YOUR_LEX_BOT_ID`**: Replace with your Lex Bot ID.
  * **`YOUR_LEX_BOT_ALIAS_ID`**: Replace with your Lex Bot Alias ID.
  * **`YOUR_API_GATEWAY_INVOKE_URL`**: Replace with the Invoke URL for your API Gateway.

-----

## **PART 3: TESTING & DEPLOYMENT**

### **Step 3.1: Local Testing**

Test the chatbot locally before deploying to ensure all AWS configurations are correct.

1.  Open a terminal or command prompt.
2.  Navigate to the directory where you saved `index.html`.
3.  Run a simple Python HTTP server:

<!-- end list -->

```bash
python -m http.server 8000
```

4.  Open a web browser and navigate to: `http://localhost:8000/index.html`
5.  Test the chatbot's functionality, including asking questions that trigger the escalation process.

### **Step 3.2: Deploying to Amazon S3**

To make the chatbot publicly accessible, you can host it using S3 Static Website Hosting.

1.  Navigate to the **Amazon S3** console.
2.  **Create a new S3 bucket**. The bucket name must be globally unique.
3.  In the bucket's **Properties** tab, find "Static website hosting" and enable it. Set `index.html` as the "Index document".
4.  In the **Permissions** tab:
      * **Disable "Block all public access"**.
      * Add a **Bucket Policy** to allow public reads (`s3:GetObject`).
5.  In the **Objects** tab, upload your configured `index.html` file.
6.  Use the **Bucket website endpoint URL** from the "Static website hosting" section to access your live chatbot.

-----

## **PART 4: USAGE AND TROUBLESHOOTING**

### **Step 4.1: How to Use the Chatbot**

  * Access the chatbot via the local URL or the S3 bucket endpoint.
  * Type an IT support question and press `Enter`.
  * The bot will attempt to answer the query. If it cannot, it will trigger the escalation process via email.
  * Use the sun/moon icon in the header to toggle the theme.

### **Step 4.2: Common Troubleshooting Tips**

  * **Chatbot is blank or not responding?**
      * Open the browser's Developer Console (F12) and check the "Console" tab for JavaScript errors.
  * **Getting a CORS error?**
      * Verify the CORS settings in **API Gateway** (Step 1.3) and ensure your frontend origin is allowed.
      * Check **Amazon CloudWatch** logs for your Lambda function to see if the function is failing.
  * **Made changes but don't see them on the S3 website?**
      * Clear your browser's cache or use an Incognito/Private window. This forces the browser to download the latest `index.html` file.
  * **Email escalation is not working?**
      * Ensure the `SENDER_EMAIL` and `RECIPIENT_EMAIL` in your Lambda code are verified in SES (Step 1.1) and that your Lambda role has `AmazonSESFullAccess` (Step 1.2).
      * Check the API Gateway Invoke URL in your `index.html` file.
