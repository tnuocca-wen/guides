### Permanent OUTBOUND IP (for Whitelisting)
**Goal:** Ensure every request leaving your Azure Function comes from one specific, unchanging Public IP address.

#### **Supported Hosting Plans**
*   **Flex Consumption:** (Newest, most cost-effective for scaling).
*   **Elastic Premium (EP):** (High performance, pre-warmed instances).
*   **Dedicated (App Service Plan):** (Basic, Standard, Premium v2/v3).
*   *Note: The "Consumption" (Y1) plan is **not** supported.*

---

#### **Step 1: Create a Static Public IP Address**
1.  In the Azure Portal, search for **Public IP addresses**.
2.  Click **+ Create**.
3.  **SKU:** Select **Standard** (Required for NAT Gateway).
4.  **Tier:** Regional.
5.  **IP Address Assignment:** Select **Static**.
6.  **Name:** `func-outbound-static-ip`.
7.  Click **Review + Create**, then **Create**.

#### **Step 2: Create the NAT Gateway**
1.  Search for **NAT gateways** and click **+ Create**.
2.  **Name:** `func-nat-gateway`.
3.  **Region:** Must be the **same region** as your Function App.
4.  **Outbound IP tab:** Select the Static IP you created in Step 1.
5.  Click **Review + Create**, then **Create**.

#### **Step 3: Configure the VNet and Subnet (Crucial Step)**
To avoid the "Grayed Out" error in the portal, your subnet must meet these specific criteria:
1.  Navigate to your **Virtual Network** (or create a new one).
2.  Select **Subnets** > **+ Subnet**.
3.  **Name:** `snet-func-integrated`.
4.  **Address Range:** 
    *   **Flex Consumption:** Requires at least a **`/28`** (16 IPs).
    *   **Premium/Dedicated:** Requires at least a **`/29`** (8 IPs).
    *   *Recommendation:* Use a **`/26`** (64 IPs) to allow for future scaling.
5.  **Subnet Delegation:** Select **`Microsoft.Web/serverFarms`**.
6.  **NAT Gateway:** Select the `func-nat-gateway` you created in Step 2.
7.  Click **Save**.

#### **Step 4: Enable VNet Integration on the Function App**
1.  Go to your **Function App** in the Azure portal.
2.  On the left menu, under **Settings**, select **Networking**.
3.  Under **Outbound Traffic**, select **VNet integration**.
4.  Click **Add VNet integration**.
5.  Select your Virtual Network and the subnet (`snet-func-integrated`) from Step 3.
6.  Click **Connect/Apply**.

#### **Step 5: Configure Routing (Plan-Specific Requirements)**
This is where the configuration differs depending on your plan:

*   **For Flex Consumption:** 
    *   **No action needed.** Flex Consumption automatically routes all internet-bound traffic through the VNet by default.
*   **For Premium (EP) or Dedicated Plans:**
    *   By default, these plans only route "Private" traffic (internal IPs) through the VNet. You must force "Internet" traffic through it.
    *   Go to **Configuration** (under Settings) in your Function App.
    *   Click **+ New application setting**.
    *   **Name:** `WEBSITE_VNET_ROUTE_ALL`
    *   **Value:** `1`
    *   Click **Save**.

---

#### **Step 6: Verification (Test the Static IP)**
1.  In the Function App portal, go to **Development Tools** > **Console** (or **SSH** for Linux).
2.  Type the following command: `curl https://ifconfig.me`
3.  **The Result:** The IP address returned must match the **Static Public IP** from Step 1.
4.  Wait 1 minute and run it again. It should remain identical.

---

### **Summary Table: Outbound Static IP**

| Feature | Flex Consumption | Premium / Dedicated |
| :--- | :--- | :--- |
| **Min Subnet Size** | `/28` | `/29` |
| **Subnet Delegation** | `Microsoft.Web/serverFarms` | `Microsoft.Web/serverFarms` |
| **Route All Traffic** | Automatic (Default) | Requires `WEBSITE_VNET_ROUTE_ALL=1` |
| **Fixed IP Source** | NAT Gateway | NAT Gateway |
