# Support Query System – n8n Workflow Documentation

## Overview

This workflow is designed to automate support query routing and notification using n8n. It classifies incoming support requests as either "customer" or "admin", fetches the appropriate users from an API, selects the best user(s) to notify, and sends an email through Gmail. The workflow includes robust error handling and dynamic response formatting.

---

## Workflow Steps

### 1. **Support Query Trigger**
- **Node:** `Support Query System`
- **Type:** Chat Trigger
- **Function:** Receives incoming support queries from users.
- **Input Example:**
  ```json
  {
    "sessionId": "abc123",
    "action": "sendMessage",
    "chatInput": "I was charged twice for my subscription. Can someone help with a refund?"
  }
  ```

### 2. **Text Classifier**
- **Node:** `Text Classifier`
- **Type:** AI Text Classifier (Gemini 2.5 Flash-Lite)
- **Function:** Classifies the incoming query into:
  - `customer`: billing, account, payment, subscription, refund, feature request, general support, etc.
  - `admin`: technical issues, system outages, API failures, urgent/critical problems, security, infrastructure, etc.
  - Uses clear category descriptions for high accuracy.

### 3. **HTTP Request – Fetch Users**
- **Node(s):** `HTTP Request` (customer) / `HTTP Request1` (admin)
- **Type:** HTTP GET
- **Function:** Fetches all users from the API:  
  `https://api.escuelajs.co/api/v1/users`
- **API Response Example:** See "User API Response Example" below.

### 4. **User Selection**
- **Node(s):** `Code` (customer), `Code1` (admin)
- **Type:** Code (Run Once for All Items)
- **Function:**  
  - For `customer`: Selects a random, active customer support user.
  - For `admin`: Selects a random admin user, or (optionally) notifies all for urgent queries.
  - Handles node name references robustly.
- **Sample Code (Customer):**
  ```javascript
  const originalQuery = $('Support Query System').first().json.chatInput;
  const users = $input.all().map(item => item.json);
  const customerUsers = users.filter(user => user.role === 'customer');
  const activeCustomerUsers = customerUsers.filter(user => user.email && user.name && user.email !== 'a@gmail.com');
  const selectedUser = activeCustomerUsers[Math.floor(Math.random() * activeCustomerUsers.length)] || customerUsers[0];
  return [{
    json: {
      selectedUser,
      userEmail: selectedUser.email,
      userName: selectedUser.name,
      userId: selectedUser.id,
      userRole: selectedUser.role,
      userAvatar: selectedUser.avatar,
      originalQuery,
      classification: 'customer',
      urgency: 'normal',
      currentDateTime: new Date().toISOString(),
      currentUser: 'rkj180220',
      ticketId: `CST-${Date.now()}`
    }
  }];
  ```
- **Sample Code (Admin):**
  ```javascript
  const originalQuery = $('Support Query System').first().json.chatInput;
  const users = $input.all().map(item => item.json);
  const adminUsers = users.filter(user => user.role === 'admin');
  const urgentKeywords = ['urgent', 'critical', 'down', 'emergency', 'production', 'api', 'system', 'outage', 'error'];
  const isUrgent = urgentKeywords.some(keyword => originalQuery.toLowerCase().includes(keyword));
  const selectedUser = adminUsers[Math.floor(Math.random() * adminUsers.length)] || adminUsers[0];
  return [{
    json: {
      selectedUser,
      userEmail: selectedUser.email,
      userName: selectedUser.name,
      userId: selectedUser.id,
      userRole: selectedUser.role,
      userAvatar: selectedUser.avatar,
      originalQuery,
      classification: 'admin',
      urgency: isUrgent ? 'critical' : 'high',
      currentDateTime: new Date().toISOString(),
      currentUser: 'rkj180220',
      ticketId: `ADM-${Date.now()}`
    }
  }];
  ```

### 5. **Send Gmail Email**
- **Node(s):** `Create a draft` (customer), `Create a draft admin` (admin)
- **Type:** Gmail
- **Function:**  
  - Sends a formatted HTML email to the selected user(s) with all ticket details.
  - Uses dynamic data from previous steps for personalization and clear instructions.
- **Subject/Body:**  
  - Customer: "New Customer Support Query - [Ticket ID] - [Date]"
  - Admin: "🚨 URGENT: Technical Query - [Ticket ID] - IMMEDIATE ATTENTION REQUIRED"
  - See code sections above for full HTML template examples.

### 6. **Response Formatter**
- **Node:** `Response Formatter`
- **Type:** Code (Run Once for Each Item)
- **Function:**  
  - Formats a user-facing response summarizing ticket creation and next steps.
  - Handles missing or undefined fields robustly.
- **Sample Code:**
  ```javascript
  const classification = $json.classification || 'unknown';
  const userEmail = $json.userEmail || 'Unknown';
  const userName = $json.userName || 'Unknown';
  const ticketId = $json.ticketId || 'Unknown';
  const urgency = $json.urgency || 'normal';
  let originalQuery = $json.originalQuery;
  if (typeof originalQuery !== 'string' || !originalQuery.length) {
    try {
      originalQuery = $('Support Query System').first().json.chatInput || 'N/A';
    } catch (e) {
      originalQuery = 'N/A';
    }
  }
  const now = new Date().toISOString().replace('T', ' ').substring(0, 19) + " UTC";
  let responseMessage;
  if (classification === 'customer') {
    responseMessage = `✅ **Customer Support Query Successfully Processed**
🎫 **Ticket Created:** ${ticketId}
👤 **Assigned to:** ${userName}
📧 **Email:** ${userEmail}
🎯 **Classification:** Customer Support  
⏱️ **Expected Response:** 2-4 hours
📅 **Submitted:** ${now}
👤 **Submitted by:** rkj180220
**Your Query:** "${originalQuery.substring(0, 100)}${originalQuery.length > 100 ? '...' : ''}"
✨ Our customer support team has been notified and will contact you soon!`;
  } else if (classification === 'admin') {
    responseMessage = `🚨 **URGENT: Technical Query Escalated**
🎫 **Emergency Ticket:** ${ticketId}
👨‍💻 **Assigned to:** ${userName} (Technical Team)
📧 **Email:** ${userEmail}
🎯 **Classification:** Technical/Admin Support
⚡ **Priority:** ${urgency.toUpperCase()} - IMMEDIATE ATTENTION
⏱️ **Expected Response:** ${urgency === 'critical' ? '15-30 minutes' : '1 hour'}
📅 **Submitted:** ${now}
👤 **Submitted by:** rkj180220
**Your Query:** "${originalQuery.substring(0, 100)}${originalQuery.length > 100 ? '...' : ''}"
⚡ Our technical team will address this ${urgency === 'critical' ? 'IMMEDIATELY' : 'urgently'}!`;
  } else {
    responseMessage = `❓ **Query Under Manual Review**
🎫 **Review Ticket:** ${ticketId}
👥 **Assigned for Review:** ${userName}
📧 **Email:** ${userEmail}
🎯 **Status:** Manual Classification Required
⏱️ **Expected Response:** 1-2 hours  
📅 **Submitted:** ${now}
👤 **Submitted by:** rkj180220
**Your Query:** "${originalQuery.substring(0, 100)}${originalQuery.length > 100 ? '...' : ''}"
🔍 We'll manually classify and route your query to the appropriate team shortly.`;
  }
  return { response: responseMessage };
  ```

---

## User API Response Example

```json
[
  {
    "id": 1,
    "email": "john@mail.com",
    "name": "Jhon",
    "role": "customer",
    "avatar": "https://i.imgur.com/LDOO4Qs.jpg"
  },
  {
    "id": 3,
    "email": "admin@mail.com",
    "name": "Admin",
    "role": "admin",
    "avatar": "https://i.imgur.com/yhW6Yw1.jpg"
  }
  // ... more users
]
```

---

## Sample Test Cases

Paste these as input to the `Support Query System` node for testing.

```json
[
  {
    "sessionId": "1a2b3c4d",
    "action": "sendMessage",
    "chatInput": "I forgot my password and can't log in to my account. Please help."
  },
  {
    "sessionId": "2b3c4d5e",
    "action": "sendMessage",
    "chatInput": "URGENT: Our website is returning a 500 error for all users."
  },
  {
    "sessionId": "3c4d5e6f",
    "action": "sendMessage",
    "chatInput": "How do I update my payment method for my subscription?"
  },
  {
    "sessionId": "4d5e6f7g",
    "action": "sendMessage",
    "chatInput": "The API is down and our integration is failing. This is impacting customers."
  },
  {
    "sessionId": "5e6f7g8h",
    "action": "sendMessage",
    "chatInput": "Can I get a refund for my last month's bill? There was an incorrect charge."
  },
  {
    "sessionId": "6f7g8h9i",
    "action": "sendMessage",
    "chatInput": "I'm getting a security warning when trying to access my dashboard. Is there a breach?"
  },
  {
    "sessionId": "7g8h9i0j",
    "action": "sendMessage",
    "chatInput": "I'd like to know more about your enterprise pricing options."
  },
  {
    "sessionId": "8h9i0j1k",
    "action": "sendMessage",
    "chatInput": "The database server is unreachable and orders are not being processed. Please fix ASAP."
  },
  {
    "sessionId": "9i0j1k2l",
    "action": "sendMessage",
    "chatInput": "I want to change my email address associated with my account."
  },
  {
    "sessionId": "0j1k2l3m",
    "action": "sendMessage",
    "chatInput": "There is a bug in the recent update causing data loss. This needs immediate attention."
  }
]
```

---

### **Expected Classifications**

| Query | Expected Classification | Notes |
|-------|------------------------|-------|
| Forgot password | customer | Account issue |
| Website 500 error (URGENT) | admin | Technical, urgent |
| Payment method update | customer | Account/billing |
| API is down, integration failing | admin | Technical, urgent |
| Refund for incorrect charge | customer | Billing/refund |
| Security warning, possible breach | admin | Security, urgent |
| Enterprise pricing options | customer | Sales inquiry |
| Database server unreachable, orders not processed | admin | Technical, urgent |
| Change email address | customer | Account |
| Bug causing data loss, immediate attention | admin | Technical, urgent |

---

## Troubleshooting & Best Practices

- **Node Name References:** Always use the correct node name in your code blocks, e.g. `$('Support Query System')`.
- **Run Mode:** For selecting from all users, set Code nodes to **Run Once for All Items**.
- **Error Handling:** Always provide fallback/default values to prevent undefined errors.
- **Expressions:** Use the Expression Editor to select fields/nodes to avoid typos.
- **User Filtering:** Adjust filtering logic as needed for your business rules (e.g., avoid test/demo accounts).

---

## Extending the Workflow

- **Multi-user notifications:** For critical admin queries, you can easily send emails to all admin users instead of just one.
- **Ticket Logging:** Integrate with a ticketing system (like Zendesk or Jira) for full lifecycle management.
- **Slack/Teams Integration:** Add notifications to chat platforms.
- **Analytics:** Log query volume, response times, and user performance.

---

## Credits

**Author:** Ramkumar Jayakumar (rkj180220)  
**Date:** 2025-07-01  
**Platform:** n8n + Gemini API + Gmail
