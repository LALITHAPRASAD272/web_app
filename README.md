# Microsoft Azure — Path-Based Routing
### Complete Setup Guide | From Zero to Working Application Gateway

---

## What You Will Learn

By the end of this guide, you will have a working Azure Application Gateway that routes traffic to two different App Services based on URL path.

- Create and deploy two Flask apps (Frontend + API) on Azure App Services
- Connect GitHub to Azure for automatic deployments
- Configure an Application Gateway with Path-Based Routing
- Set up Health Probes, Backend Pools, and Routing Rules
- Troubleshoot common 502 errors

---

## Final Architecture (What We Are Building)

```
Internet
    |
Public IP
    |
Application Gateway
    |
--------------------------
|                        |
/                    /prasad*
|                        |
Frontend App          API App
```

| User Visits | Goes To |
|---|---|
| `http://<your-public-ip>/` | Frontend App Service |
| `http://<your-public-ip>/prasad` | Backend API App Service |

> 💡 The Application Gateway acts like a traffic cop — it reads the URL and decides which App Service to send the request to.

---

## Phase 1: Create Your Two Flask Apps

### 1.1 What Is a Flask App?
Flask is a lightweight Python framework to build web applications. We need two apps:
- **Frontend App** — shows a webpage when you visit `/`
- **Backend API App** — returns data when you visit `/prasad`

### 1.2 Create Frontend App (`app.py`)

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return 'Hello from Frontend!'

app.run()
```

### 1.3 Create Backend API App

Create a **separate folder** for the API. Same structure but different route:

```python
from flask import Flask
app = Flask(__name__)

@app.route('/prasad')
def api():
    return 'Hello from API!'

app.run()
```

> 💡 Keep these two apps in **separate GitHub repositories**. Each app gets its own App Service.

---

## Phase 2: Deploy Apps to Azure App Service

### 2.1 Create App Service for Frontend

Go to **portal.azure.com** and follow these steps:

**Step 1:** Search `App Services` in the top search bar and click it.

**Step 2:** Click the `+ Create` button.

**Step 3:** Fill in the settings:

| Field | Value |
|---|---|
| Subscription | Select your Azure subscription |
| Resource Group | Create new (e.g., `myResourceGroup`) |
| Name | `frontend-app` (must be globally unique) |
| Runtime stack | Python 3.11 |
| Region | East US (or any region close to you) |
| Pricing Plan | Free F1 (for learning/testing) |

**Step 4:** Click **Review + Create** → **Create**. Wait ~2 minutes.

### 2.2 Repeat for Backend API App

Repeat all steps above but change the Name to: `api-app` (or any unique name).

> 💡 You will now have **TWO App Services** running — one for frontend, one for API.

---

## Phase 3: Connect GitHub to Azure (Deployment Center)

### 3.1 Why Use Deployment Center?
Every time you push code to GitHub, Azure automatically updates your app. No manual uploads needed.

### 3.2 Setup for Frontend App

**Step 1:** Go to App Services → click your **frontend app**.

**Step 2:** In the left sidebar, click **Deployment Center**.

**Step 3:** Under `Source`, select **GitHub**.

**Step 4:** Click **Authorize** and log in to your GitHub account.

**Step 5:** Choose your Organization, Repository (frontend repo), and Branch (`main`).

**Step 6:** Click **Save**. Azure will now auto-deploy on every push.

> 💡 After saving, go to the **Logs** tab in Deployment Center to watch the first deployment in real time.

### 3.3 Repeat for Backend API App
Do the same steps for your API App Service, but select your **API GitHub repository**.

### 3.4 Test Your Apps Directly

Before setting up the gateway, test both apps:

| App | URL | Expected Result |
|---|---|---|
| Frontend | `https://frontend-app.azurewebsites.net/` | `Hello from Frontend!` |
| API | `https://api-app.azurewebsites.net/prasad` | `Hello from API!` |

> ⚠️ If either app does not work here, **fix it BEFORE** proceeding to Application Gateway setup.

---

## Phase 4: Create the Application Gateway

### 4.1 What Is an Application Gateway?
An Application Gateway is a Layer 7 (HTTP/HTTPS) load balancer. It routes traffic to different backends based on URL path, domain name, or custom rules.

### 4.2 Create the Gateway

Search `Application Gateways` in Azure Portal → click `+ Create`.

**Basics Tab:**

| Field | Value |
|---|---|
| Resource Group | Same as your App Services |
| Name | `my-app-gateway` |
| Region | Same region as your App Services |
| Tier | Standard V2 |
| Autoscaling | Yes (min: 0, max: 2) |

**Frontends Tab:**

| Field | Value |
|---|---|
| Frontend IP type | Public |
| Public IP address | Create new → name it `gateway-public-ip` |

**Backends Tab — Create TWO Backend Pools:**

| Pool Name | Target Type | Target Value |
|---|---|---|
| `frontend-pool` | App Service | `frontend-app.azurewebsites.net` |
| `api-pool` | App Service | `api-app.azurewebsites.net` |

---

## Phase 5: Configure Health Probes

### 5.1 What Is a Health Probe?
A Health Probe regularly checks if your backend apps are alive. If an app does not respond, the gateway stops sending traffic to it.

### 5.2 Create Health Probes

Go to your Application Gateway → **Health Probes** → `+ Add`.

**Health Probe for Frontend:**

| Field | Value |
|---|---|
| Name | `frontend-probe` |
| Protocol | HTTP |
| Host | `frontend-app.azurewebsites.net` |
| Path | `/` |
| Interval | 30 seconds |
| Timeout | 30 seconds |
| Unhealthy threshold | 3 |

**Health Probe for API:**

| Field | Value |
|---|---|
| Name | `api-probe` |
| Protocol | HTTP |
| Host | `api-app.azurewebsites.net` |
| Path | `/prasad` |
| Interval | 30 seconds |
| Timeout | 30 seconds |
| Unhealthy threshold | 3 |

> 💡 Always click **Test** before saving. It should return `Status 200 OK`.

---

## Phase 6: Configure Backend Settings

### 6.1 What Are Backend Settings?
Backend Settings tell the Application Gateway HOW to talk to your backend apps — protocol, port, and host header.

### 6.2 Why Host Header Override Matters
Azure App Services only respond to requests with the correct host header. Without this override, you will get **502 errors**.

### 6.3 Create Backend Settings

Go to Application Gateway → **Backend Settings** → `+ Add`.

**Settings for Frontend:**

| Field | Value |
|---|---|
| Name | `frontend-settings` |
| Backend protocol | HTTP |
| Backend port | 80 |
| Override with new host name | **Yes** |
| Host name override | Pick from backend target → select `frontend-app` |
| Cookie based affinity | Disabled |
| Custom probe | `frontend-probe` |

**Settings for API:**

| Field | Value |
|---|---|
| Name | `api-settings` |
| Backend protocol | HTTP |
| Backend port | 80 |
| Override with new host name | **Yes** |
| Host name override | Pick from backend target → select `api-app` |
| Cookie based affinity | Disabled |
| Custom probe | `api-probe` |

---

## Phase 7: Set Up Path-Based Routing Rules

### 7.1 What Are Routing Rules?
Routing Rules define: **IF** the URL path matches X → **THEN** send traffic to backend pool Y.

### 7.2 Create the Routing Rule

**Step 1:** Go to your Application Gateway → click **Rules** → `+ Add routing rule`.

**Step 2:** Fill in Listener settings:

| Field | Value |
|---|---|
| Rule name | `path-routing-rule` |
| Priority | 100 |
| Listener name | `http-listener` |
| Frontend IP | Public |
| Protocol | HTTP |
| Port | 80 |

**Step 3:** Click the **Backend targets** tab:

| Field | Value |
|---|---|
| Target type | Backend pool |
| Backend target (default) | `frontend-pool` |
| Backend settings (default) | `frontend-settings` |

**Step 4:** Click `+ Add multiple targets` to add the path rule for API:

| Field | Value |
|---|---|
| Path | `/prasad*` |
| Target name | `api-path-rule` |
| Backend target | `api-pool` |
| Backend settings | `api-settings` |

> 💡 The `*` after `/prasad` means it matches `/prasad` and anything after it like `/prasad/data`.

**Step 5:** Click **Add** → **Save**. Wait 2–3 minutes for changes to apply.

---

## Phase 8: Test Your Setup

### 8.1 Find Your Public IP
Go to Application Gateway → **Overview** → copy the **Frontend public IP address**.

### 8.2 Test Both Routes

| Test | URL | Expected Result |
|---|---|---|
| Frontend | `http://<your-public-ip>/` | `Hello from Frontend!` |
| API | `http://<your-public-ip>/prasad` | `Hello from API!` |

✅ If both work — **Path-Based Routing is fully working!**

---

## Phase 9: Troubleshooting — 502 Bad Gateway

### 9.1 What Is a 502 Error?
502 means Application Gateway could not get a valid response from the backend. Common causes:

- Host header not set correctly in Backend Settings
- Health Probe is failing (app not running)
- Wrong backend pool target (typo in app service URL)
- App Service is stopped or crashed

### 9.2 How to Fix

**Step 1 — Check Backend Health:**
Go to Application Gateway → **Backend Health**. All pools should show `Healthy` in green.

**Step 2 — Check Host Header Override:**
Go to Backend Settings. Make sure **Override with new hostname** is set to `YES`.

**Step 3 — Test Health Probe:**
Go to Health Probes → click your probe → **Test**. Should return `200 OK`.

**Step 4 — Test App Service Directly:**
Visit `https://frontend-app.azurewebsites.net` directly. If this fails, the problem is with the app, not the gateway.

**Step 5 — Check App Service Status:**
Go to App Service → Overview. Status should show `Running`. If Stopped, click **Start**.

---

## What You Completed ✅

- [x] Azure App Services — created and deployed two Flask apps
- [x] GitHub Deployment Center — connected for auto-deploy
- [x] Application Gateway — created with public IP
- [x] Backend Pools — one for frontend, one for API
- [x] Backend Settings — with host header override
- [x] Health Probes — monitoring both apps
- [x] Path-Based Routing — `/` to frontend, `/prasad` to API
- [x] 502 Troubleshooting — know how to debug

---

## Next Level: What to Try 🚀

1. **Azure Custom Domain + SSL** — add your own domain and HTTPS certificate
2. **Azure WAF (Web Application Firewall)** — protect your apps from attacks
3. **Azure Autoscaling** — automatically scale when traffic increases
4. **Azure Traffic Manager** — route traffic across multiple regions globally
5. **Azure Front Door** — CDN + global load balancer for production apps

---

> 💡 This hands-on setup is an excellent talking point in interviews. Be ready to explain each component and why it is needed.
