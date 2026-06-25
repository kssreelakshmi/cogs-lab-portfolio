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

I deployed Keycloak using Docker:

```bash
docker run -d --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=Admin@Lab123 \
  quay.io/keycloak/keycloak:23.0.0 start-dev
```

### Issues I ran into and how I fixed them

**Issue 1 — "HTTPS required" error on admin console:**
Keycloak 23 enforces HTTPS by default even in dev mode. Clicking Administration Console showed "We are sorry... HTTPS required" and blocked access completely. I had to disable it using the `kcadm.sh` CLI inside the container:

```bash
docker exec keycloak /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 --realm master \
  --user admin --password Admin@Lab123

docker exec keycloak /opt/keycloak/bin/kcadm.sh update realms/master -s sslRequired=NONE
docker exec keycloak /opt/keycloak/bin/kcadm.sh update realms/instasafe-lab -s sslRequired=NONE
```

Note: The `-it` flag caused the command to hang indefinitely. Removing it and running without interactive mode fixed this.

**Issue 2 — OCI double firewall:**
Opening port 8080 in OCI's Security List was not enough. The VM itself had iptables rules blocking the port. I had to also run:
```bash
sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 9090 -j ACCEPT
```

**Issue 3 — admin-cli direct access grants disabled:**
The lab guide says to get a JWT token using admin credentials. In Keycloak 23, `admin-cli` has direct access grants disabled by default. The admin user also only exists in the `master` realm, not `instasafe-lab`. I enabled direct access grants for the `account` client in `instasafe-lab` and used `testuser` credentials instead to get the JWT token — same concept, just a different client.

**Issue 4 — SAML "Invalid requester" error:**
The initial Flask SP script sent a minimal `<AuthnRequest/>` which Keycloak rejected. I rewrote it with a proper deflate-compressed SAMLRequest and also disabled "Client signature required" in the SAML client Keys tab. After both fixes, the full SAML flow completed successfully.

**instasafe-lab realm dashboard:**
![instasafe-lab realm](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-2-1-image.png)

I created `testuser` (email: `testuser@instasafe.local`), set password `TestUser@123`, created the `support-team` group, and added testuser to it.

**testuser added to support-team group:**
![testuser in group](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-2-2-image.png)

---

## Part 2 — SAML Client Configured

I created a SAML client simulating InstaSafe as the Service Provider:

- **Client type:** SAML
- **Client ID:** `https://sp.instasafe.local/saml`
- **Name:** InstaSafe SP Simulation
- **Valid Redirect URI:** `http://140.245.218.252:9090/saml/callback`
- **Master SAML Processing URL:** `http://140.245.218.252:9090/saml/callback`

**Clients list showing InstaSafe SP Simulation as SAML type:**
![SAML client](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-2-3-image.png)

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

**Both mappers configured on the dedicated scope:**
![attribute mappers](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-2-4-image.png)

---

## Part 4 — IdP Metadata XML and SAML Flow Test

I downloaded the IdP metadata XML that would be given to InstaSafe during real SSO setup:

```bash
curl http://140.245.218.252:8080/realms/instasafe-lab/protocol/saml/descriptor \
  -o keycloak-idp-metadata.xml
```

The metadata contains the signing certificate, SingleSignOnService endpoints, SingleLogoutService endpoints, and NameID formats — everything InstaSafe needs to trust this Keycloak instance as an IdP.

**IdP metadata XML downloaded via curl:**
![IdP metadata](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-2-5-image.png)

I then ran a Python Flask SP simulation to complete the full SAML flow:

1. Browser hits `http://140.245.218.252:9090/saml/login`
2. Flask builds a proper deflate-compressed SAMLRequest and redirects to Keycloak
3. Keycloak shows the INSTASAFE-LAB login page
4. I logged in as `testuser`
5. Keycloak posted a SAMLResponse back to `/saml/callback`
6. Flask decoded it and displayed the user attributes

**Decoded SAML response showing user attributes:**
![SAML response decoded](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-2-6-image.png)

The decoded response contained:
- **Issuer:** `http://140.245.218.252:8080/realms/instasafe-lab`
- **Username:** `testuser`
- **Email:** `testuser@instasafe.local`
- **Group:** `/support-team`
- **Roles:** `default-roles-instasafe-lab`, `offline_access`

This proves the full SSO flow — authentication and attribute passing — works end to end.

---

## SSO Troubleshooting: What to Check When a Customer Reports "Attribute Error"

When a customer says users are logging in via SSO but getting an Attribute Error in InstaSafe, it almost always means the SAML response is missing an expected attribute — usually email or group membership.

Here is exactly what I would check, in order:

**Step 1 — Check the attribute mappers exist**
Go to: `instasafe-lab realm → Clients → [SP client] → Client scopes tab → click the dedicated scope → Mappers tab`

If the email or groups mapper is missing, the SAML response won't contain those attributes at all. This is the most common cause — someone set up the SAML client but forgot to add mappers.

**Step 2 — Check mapper attribute names**
Click each mapper and verify the SAML Attribute Name matches exactly what InstaSafe expects — it's case-sensitive. If Keycloak sends `Email` but InstaSafe expects `email`, it won't match.

**Step 3 — Check the user has the attribute set**
Go to: `instasafe-lab realm → Users → click the user → Details tab`

If a user was created without an email, the email mapper sends an empty attribute even though the mapper is correctly configured.

**Step 4 — Check group membership**
Go to: `instasafe-lab realm → Users → click the user → Groups tab`

If the groups mapper is configured but the user isn't in any group, the group attribute in the SAML response will be empty.

**Step 5 — Capture and decode the raw SAML response**
Use the SAML-tracer browser extension to capture the raw SAMLResponse. Decode it from base64 and look at the `<saml:Attribute>` elements. This shows exactly what Keycloak is sending vs what InstaSafe expects — fastest way to diagnose any mismatch.

Attribute errors in SSO are almost never a network issue. They're either a missing mapper, a wrong attribute name, or a user who doesn't have the attribute set in the IdP.

---

## Keycloak → InstaSafe SSO Component Mapping

| Keycloak Component | InstaSafe SSO Equivalent | What it does |
|---|---|---|
| **Realm (`instasafe-lab`)** | **Customer's SSO tenant** | An isolated identity domain. Each customer gets their own SSO config in InstaSafe, just like each realm in Keycloak is completely separate. |
| **SAML Client (`https://sp.instasafe.local/saml`)** | **InstaSafe as Service Provider** | The client represents the app receiving the SAML assertion. In a real setup, this is InstaSafe's SP entity ID registered in the customer's IdP. |
| **Client ID / Entity ID** | **SP Entity ID in InstaSafe SSO config** | Unique identifier for InstaSafe in the customer's IdP. If this doesn't match exactly, the IdP rejects the authentication request. |
| **Attribute Mapper (email)** | **Email attribute mapping in InstaSafe** | Tells Keycloak to include the user's email in the SAML response. InstaSafe uses this to identify and provision the user on first login. |
| **Attribute Mapper (groups)** | **Group/role sync in InstaSafe** | Passes group membership to InstaSafe so users get the right access policies based on their IdP groups. |
| **IdP Metadata XML** | **Metadata file uploaded during SSO setup** | The file a customer uploads to InstaSafe when configuring SSO. Contains the signing certificate and SSO endpoint URLs. |
| **X509 Signing Certificate** | **IdP certificate in InstaSafe trust store** | InstaSafe uses this to verify SAML responses came from the customer's IdP and haven't been tampered with. |
| **SingleSignOnService URL** | **SSO URL in InstaSafe configuration** | The endpoint InstaSafe redirects users to when they click "Sign in with SSO". |
| **testuser login flow** | **End user SSO login in InstaSafe** | Full flow: user clicks SSO → redirected to IdP → authenticates → SAML response sent back → InstaSafe grants access based on attributes. |

---

## What I Learned

**Keycloak 23 has stricter security defaults than older versions.** The HTTPS enforcement and disabled direct access grants are improvements, but they break lab guides written for older versions. I had to research and fix each one.

**Cloud environments have multiple firewall layers.** OCI Security List + VM iptables — both need to be open. Not knowing this cost me time during the WireGuard lab too.

**SAML has a lot of moving parts.** Entity ID, ACS URL, attribute names, signing requirements, and NameID format all have to match exactly. A single mismatch silently breaks the whole flow.

**Attribute mappers are not optional.** Without them, Keycloak authenticates the user but sends back a SAML response with no attributes. The SP validates the signature and then has no idea who the user is.

**The IdP metadata XML is the trust document.** Everything the SP needs to trust the IdP is in that file. When a customer's SSO breaks after a certificate rotation, it's almost always because they forgot to update this file in InstaSafe.

---

## Final Checklist

| Criteria | Status |
|---|---|
| Keycloak deployed via Docker | ✅ |
| Admin console accessible — instasafe-lab realm | ✅ |
| testuser created with email | ✅ |
| support-team group created | ✅ |
| testuser added to support-team | ✅ |
| SAML client configured — type SAML confirmed | ✅ |
| Email attribute mapper added | ✅ |
| Groups attribute mapper added | ✅ |
| IdP metadata XML downloaded | ✅ |
| Full SAML flow completed — response decoded | ✅ |
| OCI-specific issues documented | ✅ |
| SSO troubleshooting answer with menu paths | ✅ |
| Keycloak → InstaSafe mapping written | ✅ |
