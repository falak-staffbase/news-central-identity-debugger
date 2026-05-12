# 🩺 sb-auth-doctor

> **Staffbase Identity Diagnostic Center for News Central**  
> An SPFx web part that diagnoses why Staffbase News Central SSO fails in SharePoint Online — without opening a support ticket.

---

## 🏆 Staffbase Hackathon 2026

**Team:** Matthias Tietz, Muhammad Ahmad Falak, Amit Nath 
**Category:** Developer Tooling / Customer Experience  
**Problem it solves:** 101+ support tickets per year caused by authentication/setup failures in News Central × SharePoint integrations.

---

## 🔍 The Problem

Customers integrating **Staffbase News Central** with **SharePoint Online** frequently hit authentication walls — especially around SSO setup. These failures are:

- Hard to debug without deep Entra + Staffbase knowledge
- Invisible to the end user ("it just doesn't load")
- Expensive for TSE teams to investigate ticket by ticket

**This tool puts the diagnostic power directly in the hands of admins.**

---

## 💡 The Solution

A guided SPFx web part that simulates and inspects the Staffbase News Central authentication flow end-to-end:

| Check | What it does |
|---|---|
| 🪪 **Identity Probe** | Reads current user identity from `pageContext`, Graph `/me`, and decoded Entra token |
| 🔑 **Token Inspector** | Decodes the SPFx Entra access token and surfaces all relevant claims (`oid`, `upn`, `email`, `preferred_username`, `tid`, etc.) |
| 🗺️ **Claim Mapping Validator** | Compares candidate identifier values against Staffbase user records via the User API |
| 👤 **Staffbase User Lookup** | Probes whether a matching user exists in Staffbase for each candidate claim value |
| 🌐 **Network Reachability Check** | Verifies the configured Staffbase Web App URL is reachable and iframe-embeddable |
| 📋 **Diagnostic Report** | Generates a one-click copyable summary for support escalation |

---

## 🏗️ Architecture

```
SharePoint Page
└── SPFx Web Part (sb-auth-doctor)
    ├── pageContext           → instant user/tenant info
    ├── MSGraphClientFactory  → authoritative user record (/me)
    ├── AadTokenProviderFactory → decoded Entra token claims
    └── AadHttpClientFactory  → calls secure backend proxy
                                      │
                              Azure Function (Easy Auth)
                                      │
                              Key Vault (Staffbase API Token)
                                      │
                              Staffbase User API
                              GET /api/users/search?filter=...
```

> ⚠️ The Staffbase API token is **never** stored in the SPFx bundle. It lives in Azure Key Vault, accessed only by the backend proxy secured with Entra ID authentication.

---

## 🔬 How Claim Mapping Verification Works

Staffbase News Central 2.0+ authenticates users via **Entra ID SAML SSO**. The SAML `NameID` emitted by the customer's Entra Enterprise Application must exactly match the `externalID` (identifier) of the Staffbase user.

This tool:
1. Reads **every possible candidate value** from Microsoft's identity surfaces (`oid`, `upn`, `mail`, `userPrincipalName`, `employeeId`, `onPremisesImmutableId`, etc.)
2. Calls the **Staffbase User API** via a secure proxy to check which of those values matches an existing Staffbase user
3. Maps the match back to the **Entra source attribute** the admin needs to set in their Enterprise Application

**Example output:**

| Candidate | Value | Found in Staffbase? | Entra Source Attribute |
|---|---|---|---|
| `userPrincipalName` | jane@contoso.com | ✅ Match | `user.userprincipalname` |
| `mail` | jane.doe@contoso.com | ❌ No match | — |
| `employeeId` | E12345 | ❌ No match | — |

---

## 🚀 Getting Started

### Prerequisites

- Node.js 18.x
- SharePoint Framework (SPFx) 1.18+
- Azure subscription (for the backend proxy function)
- Staffbase API token (created in Studio → Settings → API Access)

### Installation

```bash
git clone https://github.com/your-org/sb-auth-doctor.git
cd sb-auth-doctor
npm install
```

### Local development

```bash
gulp serve
```

### Build & deploy

```bash
gulp bundle --ship
gulp package-solution --ship
```

Then upload the `.sppkg` from `./sharepoint/solution/` to your SharePoint App Catalog.

### Required API permissions

After deploying, a tenant admin must approve the following in **SharePoint Admin → API Access**:

| API | Permission | Reason |
|---|---|---|
| Microsoft Graph | `User.Read` | Read current user's profile |
| Your Azure Function App | `user_impersonation` | Call the Staffbase proxy securely |

---

## 📁 Project Structure

```
sb-auth-doctor/
├── src/
│   └── webparts/
│       └── sbAuthDoctor/
│           ├── components/
│           │   ├── IdentityProbe/       # pageContext + Graph /me panel
│           │   ├── TokenInspector/      # Decoded Entra token claims
│           │   ├── ClaimMappingTable/   # Candidate → Staffbase match results
│           │   ├── NetworkProbe/        # Reachability checks
│           │   └── DiagnosticReport/    # Copy-to-clipboard summary
│           └── SbAuthDoctorWebPart.ts
├── azure-function/
│   └── StaffbaseUserProbe/              # Secure backend proxy
│       ├── index.ts
│       └── README.md
├── config/
│   └── package-solution.json
└── README.md
```

---

## ⚠️ Important Notes

- **Decoded tokens are unverified client-side.** The SPFx runtime uses the *SharePoint Online Client Extensibility* principal to issue tokens; their signatures cannot be validated in the browser. All decoded claim data is labelled "for display only" in the UI.
- **`email`, `upn`, and `preferred_username` are not reliable identity keys** per Microsoft's own guidance. This tool uses them for *diagnostic comparison* only, never for authorization.
- **Guest/B2B accounts** with `#EXT#@tenant.onmicrosoft.com` UPNs are a first-class diagnostic scenario and are flagged automatically.

---

## 📊 Expected Impact

| Metric | Target |
|---|---|
| Support tickets reduced | ~101/year → significant reduction |
| Time to diagnose auth failure | Hours → Minutes |
| Self-service resolution rate | Increase without TSE involvement |
| TSE team load | Reduced |

---

## 🤝 Contributing

This project was built during the Staffbase Hackathon 2026. Contributions, feedback, and issue reports are welcome.

---

## 📄 License

MIT — see [LICENSE](./LICENSE)

---

*Built with ❤️ at Staffbase Hackathon 2026*