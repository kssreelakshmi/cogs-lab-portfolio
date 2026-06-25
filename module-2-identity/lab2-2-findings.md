
# Lab 2.2 — Keycloak SSO: Build Your Own Identity Provider

## My Setup
- **Cloud Provider:** Oracle Cloud Infrastructure (OCI)
- **Region:** ap-hyderabad-1
- **OS:** Ubuntu 22.04 (vpn-server VM, Public IP: `140.245.218.252`)
- **Keycloak Version:** 23.0.0 (Docker container)
- **Realm:** `instasafe-lab`
- **SAML Client ID:** `https://sp.instasafe.local/saml`

---

## Part 1 — Keycloak Running and Admin Console Accessible

I deployed Keycloak using Docker with a single command. The first hurdle was the "HTTPS required" error when trying to access the admin console — Keycloak 23 enforces HTTPS by default even in dev mode. I had to disable SSL requirement using the `kcadm.sh` CLI tool inside the container:

```bash
docker exec keycloak /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 --realm master \
  --user admin --password Admin@Lab123

docker exec keycloak /opt/keycloak/bin/kcadm.sh update realms/master -s sslRequired=NONE
docker exec keycloak /opt/keycloak/bin/kcadm.sh update realms/instasafe-lab -s sslRequired=NONE
```

There was also a second OCI firewall layer issue — I had to open port 8080 both in the OCI Security List and via iptables on the VM itself.

Once inside, I created the `instasafe-lab` realm, added `testuser` with email `testuser@instasafe.local`, set the password, created the `support-team` group, and added testuser to it.

**Keycloak Admin Console — instasafe-lab realm:**
![Keycloak admin console](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-2-1-image.png)

**testuser created and added to support-team:**
![testuser in group](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-2-2-image.png)

---

## Part 2 — SAML Client Configured

I created a SAML client simulating InstaSafe as the Service Provider:

- **Client type:** SAML
- **Client ID:** `https://sp.instasafe.local/saml`
- **Name:** InstaSafe SP Simulation
- **Valid Redirect URI:** `http://140.245.218.252:9090/saml/callback`
- **Master SAML Processing URL:** `http://140.245.218.252:9090/saml/callback`

I also had to disable "Client signature required" under the Keys tab — this is needed when the SP doesn't sign its AuthnRequests, which is the case for our simulation script.

**SAML client settings:**
![SAML client](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-2-3-image.png)

---

## Part 3 — Attribute Mappers Configured

I added two mappers to the dedicated client scope so Keycloak includes user attributes in the SAML response:

**Mapper 1 — Email:**
- Type: User Property
- Name: `email`
- Property: `email`
- SAML Attribute Name: `email`

**Mapper 2 — Groups:**
- Type: Group list
- Name: `groups`
- Group attribute name: `groups`
- Single Group Attribute: On

**Mappers configured:**
![attribute mappers](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-2-4-image.png)

---

## Part 4 — IdP Metadata XML and SAML Flow Test

I downloaded the IdP metadata XML that would be given to InstaSafe during real SSO setup:

```bash
curl http://140.245.218.252:8080/realms/instasafe-lab/protocol/saml/descriptor \
  -o keycloak-idp-metadata.xml
```

The metadata contains the signing certificate, SingleSignOnService endpoints, SingleLogoutService endpoints, and supported NameID formats — everything InstaSafe needs to trust this Keycloak instance as an IdP.

**IdP metadata XML in browser:**
![IdP metadata](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-2-5-image.png)

I then ran a Python Flask SP simulation to complete the full SAML flow:

1. Browser hits `http://140.245.218.252:9090/saml/login`
2. Flask builds a SAMLRequest and redirects to Keycloak
3. Keycloak shows the login page (INSTASAFE-LAB realm)
4. I logged in as `testuser`
5. Keycloak sent a SAMLResponse back to `/saml/callback`
6. Flask decoded it and displayed the user attributes

**SAML response received — decoded output:**
![SAML response](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-2-6-image.png)

The decoded response contained:
- **Issuer:** `http://140.245.218.252:8080/realms/instasafe-lab`
- **Username:** `testuser`
- **Email:** `testuser@instasafe.local`
- **Group:** `/support-team`
- **Roles:** `default-roles-instasafe-lab`, `offline_access`

This proves the full SSO flow — authentication + attribute passing — works end to end.

**testuser logged into Keycloak account portal:**
![testuser logged in](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-2-7-image.png)

---

## SSO Troubleshooting: What to Check When a Customer Reports "Attribute Error"

This is one of the most common SSO support issues. When a customer says "users are logging in via SSO but getting an Attribute Error in InstaSafe", it almost always means the SAML response is missing an expected attribute — usually email or group membership.

Here is exactly what I would check in Keycloak, in order:

**Step 1 — Check the attribute mappers exist**
Go to: `instasafe-lab realm → Clients → [the SP client] → Client scopes tab → click the dedicated scope → Mappers tab`

Look for the email and groups mappers. If they're missing, the SAML response won't contain those attributes at all. This is the most common cause — someone set up the client but forgot to add mappers.

**Step 2 — Check the mapper configuration**
Click each mapper and verify:
- For email mapper: **Property** must be `email`, **SAML Attribute Name** must exactly match what InstaSafe expects (case-sensitive)
- For groups mapper: **Group attribute name** must match exactly, and **Single Group Attribute** should be On

If the attribute name in Keycloak is `Email` but InstaSafe expects `email`, it won't match and the attribute error appears.

**Step 3 — Check the user actually has the attribute set**
Go to: `instasafe-lab realm → Users → click the user → Details tab`

Check that the email field is filled in. If a user was created without an email, the email mapper will send an empty attribute even though the mapper is configured correctly.

**Step 4 — Check group membership**
Go to: `instasafe-lab realm → Users → click the user → Groups tab`

Verify the user is actually in the group. If the groups mapper is configured but the user isn't in any group, the group attribute in the SAML response will be empty.

**Step 5 — Check the SAML response directly**
Use a browser SAML tracer extension (like SAML-tracer for Chrome) to capture the raw SAMLResponse. Decode it from base64 and look at the `<saml:Attribute>` elements. This shows exactly what attributes Keycloak is sending vs what InstaSafe is expecting. This is the fastest way to diagnose a mismatch.

**Root cause summary:** Attribute errors in SSO are almost never a network issue. They're either a missing mapper, a wrong attribute name, or a user who doesn't have the attribute set in the IdP.

---

## Keycloak → InstaSafe SSO Component Mapping

| Keycloak Component | InstaSafe SSO Equivalent | What it does |
|---|---|---|
| **Realm (`instasafe-lab`)** | **Customer's SSO tenant** | A realm is an isolated identity domain. Each customer in InstaSafe gets their own SSO configuration, just like each realm in Keycloak is completely separate. |
| **SAML Client (`https://sp.instasafe.local/saml`)** | **InstaSafe as Service Provider** | The client represents the application that will receive the SAML assertion. In a real setup, this would be InstaSafe's SP entity ID registered in the customer's IdP. |
| **Client ID / Entity ID** | **SP Entity ID in InstaSafe SSO config** | This is the unique identifier for InstaSafe in the customer's IdP. If this doesn't match exactly, the IdP rejects the authentication request. |
| **Attribute Mapper (email)** | **Email attribute mapping in InstaSafe** | Tells Keycloak to include the user's email in the SAML response. InstaSafe uses this to identify and provision the user on first login. |
| **Attribute Mapper (groups)** | **Group/role sync in InstaSafe** | Passes group membership to InstaSafe so users get the right access policies automatically based on their AD/IdP groups. |
| **IdP Metadata XML** | **Metadata file uploaded during SSO setup** | This is the file a customer uploads to InstaSafe when configuring SSO. It contains the signing certificate and SSO endpoint URLs so InstaSafe knows where to send authentication requests and how to verify responses. |
| **X509 Signing Certificate** | **IdP certificate in InstaSafe trust store** | InstaSafe uses this certificate to verify that SAML responses actually came from the customer's IdP and haven't been tampered with. |
| **SingleSignOnService URL** | **SSO URL in InstaSafe configuration** | The endpoint InstaSafe redirects users to when they click "Sign in with SSO". |
| **testuser login flow** | **End user SSO login in InstaSafe** | The complete flow: user clicks SSO login → redirected to IdP → authenticates → SAML response sent back → InstaSafe grants access based on attributes. |

---

## What I Learned

**Keycloak 23 has stricter defaults than older versions.** The HTTPS requirement and disabled direct access grants are security improvements, but they break a lot of lab guides written for older versions. I had to debug each one separately.

**The SAML flow has more moving parts than it looks.** There are at least 5 things that have to match exactly: entity ID, ACS URL, attribute names, signing requirements, and NameID format. A mismatch in any one of them breaks the whole flow silently.

**Attribute mappers are not optional.** Without them, Keycloak authenticates the user fine but sends back a SAML response with no user attributes. The SP receives it, validates the signature, then has no idea who the user is because there's no email or username in the assertion.

**Docker makes Keycloak setup reproducible.** The entire setup — Keycloak, realm, users, clients — can be scripted. In a real enterprise deployment, this would all be done via Keycloak's REST API or Terraform, not the UI.

**The IdP metadata XML is the handshake document.** Everything the SP needs to trust the IdP is in that XML file — the certificate, the endpoints, the supported bindings. When a customer's SSO breaks after a certificate rotation, it's because they forgot to update this file in InstaSafe.

---

## Final Checklist

| Criteria | Status |
|---|---|
| Keycloak deployed via Docker | ✅ |
| Admin console accessible | ✅ |
| `instasafe-lab` realm created | ✅ |
| `testuser` created with email | ✅ |
| `support-team` group created | ✅ |
| testuser added to support-team | ✅ |
| SAML client configured | ✅ |
| Email attribute mapper added | ✅ |
| Groups attribute mapper added | ✅ |
| IdP metadata XML downloaded | ✅ |
| JWT access token retrieved via API | ✅ |
| Full SAML flow completed — response decoded | ✅ |
| SSO troubleshooting answer written | ✅ |
| Keycloak → InstaSafe mapping written | ✅ |
