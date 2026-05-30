An automated workflow orchestration system built with n8n that handles subscription renewals. It ensures data integrity by performing safe database updates, calculating dynamic expiry dates, and maintaining a strict historical audit trail in Airtable.

![Workflow](https://github.com/heskeyman/gym-subscription-renewal/blob/main/renewal%20workflow.JPG?raw=true)
 
Project Structure
This repository contains the n8n workflow logic and documentation for a complete renewal lifecycle:

Webhook Ingestion: Receives renewal payloads from external payment gateways or frontend forms (JSON format).
Validation & Math Engine: A JavaScript-powered Code Node that validates inputs and calculates precise expiry dates based on plan tiers (Monthly: 30d, Quarterly: 90d, Annual: 365d).
State Management (Snapshot Technique): Captures the user's pre-renewal state (e.g., "Expired" status, old expiry date) into variables before the database mutation occurs.
Safe Database Mutation: Locates the specific Airtable Record ID and updates the subscription status to "Active" without creating duplicate entries.
Audit Logging: Creates a new entry in the Renewal Logs table, explicitly documenting the transition from Previous Status → New Status.
Multi-Channel Notifications: Sends transactional HTML emails to both the renewed user (confirmation) and gym management (alert).

🛠️ Tech Stack
Automation: n8n
Database:Airtable (Members & Renewal Logs tables)
Email: Gmail / SMTP
Logic: JavaScript (ES6+ Date math & data mapping)
Testing: Postman
🚀 Key Technical Features
1. The "Snapshot" Audit Technique
Standard database updates are destructive; once an Airtable record is updated, the previous data is overwritten and lost. To maintain a permanent history of subscription changes, this workflow implements a Snapshot Pattern:

Fetch: The Find Member node retrieves the current record.
Store: An Edit Fields (Set) node immediately copies critical fields (Subscription Status, Expiry Date) into temporary variables (old_status, old_expiry).
Mutate: The Update node changes the live record to "Active" and adds time.
Log: The Create Renewal Log node reads from the Snapshot Variables (for "Before" data) and the Live Record (for "After" data) to create a perfect historical entry.
2. Dynamic Date Calculation Engine
Instead of relying on hardcoded logic, the system uses JavaScript to dynamically calculate dates based on the incoming plan_type. This ensures the system is scalable if new plans (e.g., "Weekly" or "Bi-Annual") are added later.
// Dynamic Duration Mapping
const durations = { 
    monthly: 30, 
    quarterly: 90, 
    annual: 365 
};

// Safe Date Calculation with Fallback
const plan = body.plan_type || 'monthly';
const now = new Date();
const expiry = new Date(now);

// Calculate future date
expiry.setDate(now.getDate() + (durations[plan] || 30));

// Return Airtable-compatible format (YYYY-MM-DD)
return {
    expiry_date: expiry.toISOString().split('T')[0]
};

3. Idempotent Webhook Responses
To ensure reliable integration with external payment processors (which often retry failed requests), the workflow utilizes the Respond to Webhook node. This sends an immediate 200 OK JSON response to the caller, signaling successful processing and preventing unnecessary retry loops.

📡 API Integration
To trigger the workflow manually or via a script, send a POST request to your n8n webhook URL.
ENDPOINT:POST https://<your-n8n-instance>/webhook/gym-renewal
Content-Type: application/json

Request Body: {
  "body": {
    "email": "member@example.com",
    "plan_type": "quarterly",
    "amount_paid": 45000.00,
    "transaction_id": "tx_99887766" 
  }
}

Success Response (200 OK):
{
  "success": true,
  "message": "Subscription renewed successfully",
  "data": {
    "member_id": "recXXXXXXXXXXXXXX",
    "new_expiry_date": "2026-08-27"
  }
}

 Installation & Setup
Download: Clone this repository or download the renewal-workflow.json file.
Import: Open n8n → Workflows → Add Workflow → Import from File.
Configure:
Click on each Airtable node and select your credentials (Personal Access Token required).
Verify the Base ID matches your specific Airtable URL.
Enter your gym_management_email in the Workflow Variables.
Activate: Toggle the Active switch in the top right corner to enable the production webhook URL.

 Troubleshooting
Error Message	Cause	Solution
"Unused Respond to Webhook node found"	n8n detects multiple or disconnected Respond nodes.	Ensure you have only one Respond node at the very end of the chain. Delete any duplicates.
"Invalid Formula" (Airtable)	Syntax error in the Filter by Formula field.	Use the Add Filter UI (click 'Add Filter' > select Column > equals > value) instead of writing raw formulas.
"Cannot read properties of undefined"	Data path mismatch (e.g., looking for json.email inside json.fields).	Inspect the Output of the previous node. Airtable updates usually nest data under $json.fields['Email'].
"Node not executed" (Webhook)	Webhook listener is inactive.	Click "Listen for Test Event" on the Webhook node before sending the request from Postman.

