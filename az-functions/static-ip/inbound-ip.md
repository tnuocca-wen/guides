# Configuring a Permanent INBOUND IP for Azure Functions

By default, Azure Functions are accessed via a shared, dynamic IP address. If your architecture requires a **static entry point**—for example, to map a DNS "A Record" (e.g., `api.contoso.com`) directly to an IP address or to meet strict firewall requirements for your clients—you need an **Azure Application Gateway**.

### Supported Plans
This guide applies to the following Azure Function plans:
*   **Flex Consumption** (The newest, VNet-capable consumption tier).
*   **Elastic Premium (EP)**.
*   **Dedicated (App Service Plan)**.

---

### Prerequisites
1.  An **Azure Virtual Network (VNet)**.
2.  A **Dedicated Subnet** for the Application Gateway (e.g., `snet-gateway`). 
    *   *Note: This subnet must be empty and must NOT have any Subnet Delegation.*
3.  Your **Azure Function App** already deployed.

---

### Step 1: Create a Static Public IP Address
This is the "Permanent" part of your setup. This IP address will belong to you until you manually delete it.

1.  In the Azure Portal, search for **Public IP addresses** and click **+ Create**.
2.  **SKU:** Select **Standard** (Required for the Gateway V2).
3.  **Tier:** Regional.
4.  **IP Address Assignment:** Select **Static**.
5.  **Name:** `func-inbound-static-ip`.
6.  Click **Review + Create**, then **Create**.

---

### Step 2: Deploy the Azure Application Gateway
The Application Gateway acts as the secure entry point, routing traffic from your permanent IP to your Function App.

1.  Search for **Application Gateways** and click **+ Create**.
2.  **Basics Tab:**
    *   **Tier:** Standard V2 (or WAF V2 if you need a firewall).
    *   **Virtual Network:** Select your VNet.
    *   **Subnet:** Select your dedicated gateway subnet (`snet-gateway`).
3.  **Frontends Tab:**
    *   **Frontend IP type:** Public.
    *   **Public IP address:** Select the `func-inbound-static-ip` created in Step 1.
4.  **Backends Tab:**
    *   Click **Add a backend pool**.
    *   **Target type:** Select **App Services**.
    *   **Target:** Select your specific Function App.
5.  **Configuration Tab:**
    *   **Routing Rules:** Click **Add a routing rule**.
    *   **Listener:** Name it `http-listener`, protocol **HTTP**, Port **80**.
    *   **Backend targets:** Select your Backend Pool. 
    *   **Backend Settings:** Create new. Ensure **"Override with new host name"** is set to **Yes** and **"Pick host name from backend address"** is checked. (This ensures the Gateway correctly identifies your Function App).

---

### Step 3: Secure the Function App (Crucial Step)
To ensure your permanent IP is the *only* way to reach your function, you must block direct access to the default `azurewebsites.net` URL.

1.  Navigate to your **Function App** > **Settings** > **Networking**.
2.  Under **Inbound Traffic**, select **Access Restrictions**.
3.  Click **+ Add** to create a new rule:
    *   **Name:** `Allow-Only-Gateway`.
    *   **Action:** Allow.
    *   **Priority:** 100.
    *   **Type:** Virtual Network.
    *   **Subnet:** Select the **Application Gateway Subnet** (`snet-gateway`).
4.  **The "Deny" Rule:** Ensure the "Unmatched rule action" at the bottom of the list is set to **Deny**. 
    *   *This tells Azure: "If the traffic didn't come through my Gateway, block it."*

---

### Step 4: Verification

#### 1. Test the Permanent IP
Find the **Frontend Public IP** of your Application Gateway. Open your browser or a tool like Postman and navigate to:
`http://<Your-Static-IP>/api/<Your-Function-Name>`
**Result:** Your function should execute successfully.

#### 2. Test the Default URL
Navigate to the default Azure URL:
`https://<app-name>.azurewebsites.net`
**Result:** You should receive a **403 Forbidden** error. This confirms your security configuration is working and the function is now "hidden" behind your permanent IP.

---

### Why this works for Flex, Premium, and Dedicated
In the past, basic Consumption plans could not restrict traffic based on VNets. However, **Flex Consumption** introduces the ability to use **Access Restrictions** and **Private Endpoints**. 

By placing the Application Gateway in a VNet and telling the Function App to only trust traffic from that Gateway's subnet, you create a secure, static entry point regardless of which high-performance plan you are using.
