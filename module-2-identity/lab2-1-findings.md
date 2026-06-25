# Lab 2.1 — OpenLDAP: Build Own Active Directory - Findings

## My Setup

- **OS:** Ubuntu 22.04
- **OpenLDAP version:** slapd 2.6.10
- **Base DN:** `dc=lab,dc=instasafe,dc=local`
- **Organisation:** InstaSafe Lab
- **Admin DN:** `cn=admin,dc=lab,dc=instasafe,dc=local`

---

## Part 1 — OpenLDAP Installed and Running

Installing slapd was straightforward but the `dpkg-reconfigure slapd` step is where we actually set up the directory. We have to get the DNS domain right — I set it to `lab.instasafe.local` which becomes the base DN `dc=lab,dc=instasafe,dc=local`. If we get this wrong, all subsequent queries will fail because the base DN won't match.

After configuration, slapd came up cleanly:

![slapd running](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-1-1-image.png)

Then I ran a base ldapsearch to confirm the directory structure exists:

```bash
ldapsearch -x -H ldap://localhost -b 'dc=lab,dc=instasafe,dc=local'
```

It returned the root entry with `objectClass: organization` and `o: InstaSafe Lab` — the directory is live and queryable.

![base ldapsearch](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-1-2-image.png)

---

## Part 2 — Users Created and Queryable

I created a `users.ldif` file with two organisational units (People and Groups) and two users — alice and bob. The LDIF format is how LDAP handles bulk data — every entry has a DN (its unique address in the directory tree) and a set of attributes.

Here's what I added:
- `ou=People` — the container for user accounts
- `ou=Groups` — the container for groups (used later for role-based access)
- `uid=alice` — Alice Smith, `alice@lab.instasafe.local`
- `uid=bob` — Bob Jones, `bob@lab.instasafe.local`

![users.ldif and ldapadd](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-1-3-image.png)

After adding them, I ran ldapsearch filtered by `objectClass=inetOrgPerson` to list all users with their name and email:

![ldapsearch users](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-1-4-image.png)

Both alice and bob are returned with their correct `cn` and `mail` attributes. `numEntries: 2` confirms exactly two users exist — no duplicates, no missing entries.

I also tested a search by email address — the kind of lookup InstaSafe does when a user types their email at login:

```bash
ldapsearch -x -H ldap://localhost -b 'ou=People,dc=lab,dc=instasafe,dc=local' '(mail=alice@lab.instasafe.local)' cn uid
```

It returned alice's full DN and uid correctly.

![email search](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-1-5-image.png)

---

## Part 3 — ldapwhoami Bind Test for Alice

`ldapwhoami` simulates exactly what happens when a user logs in through LDAP. It performs a bind — the client sends a DN and password to the server, and the server either accepts or rejects it. This is the core of LDAP authentication.

I ran it with Alice's correct credentials:

```bash
ldapwhoami -x -H ldap://localhost -D 'uid=alice,ou=People,dc=lab,dc=instasafe,dc=local' -w Alice@123
```

The server responded with Alice's full DN — bind successful. This means the directory confirmed Alice's identity correctly.

![ldapwhoami success](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-1-6-image.png)

---

## Part 4 — Error Code 49: Wrong Password

I ran the same bind with a wrong password:

```bash
ldapwhoami -x -H ldap://localhost -D 'uid=alice,ou=People,dc=lab,dc=instasafe,dc=local' -w wrongpassword
```

The server returned:
```
ldap_bind: Invalid credentials (49)
```

![error 49](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab2-1-7-image.png)

**What does error 49 mean and what does a support engineer do when they see it?**

Error code 49 is `LDAP_INVALID_CREDENTIALS`. It means the bind attempt was rejected — the DN was found in the directory but the password didn't match.

When a support engineer sees error 49 in an AD sync or LDAP authentication log, this is their checklist:

1. **Wrong password** — most common cause. The user may have recently changed their Active Directory password but the InstaSafe connector still has the old one cached. Fix: update the bind credentials in the InstaSafe LDAP connector config.

2. **Wrong bind DN** — the DN format might be wrong. Some AD setups expect `user@domain.com` (UPN format) while others expect `cn=username,ou=People,dc=domain,dc=com` (full DN format). Fix: check the bind DN format in the connector settings.

3. **Account locked** — if the user has too many failed login attempts in AD, the account gets locked. Error 49 is returned even if the password is correct. Fix: unlock the account in Active Directory and check the lockout policy.

4. **Bind user account expired** — if the service account used for the LDAP sync has an expired password, every sync will fail with error 49. Fix: reset the service account password and update it in the connector.

5. **Replication lag** — in large AD environments, a password change on one domain controller may not have replicated to all others yet. The connector might be hitting a DC that still has the old password. Fix: wait for replication or point the connector to the primary DC.

Error 49 is almost always a credentials issue, not a network or server issue.
---

## LDAP Concepts → InstaSafe AD Sync Mapping

This is where the lab connected everything for me. Every time InstaSafe syncs with a customer's Active Directory, it's doing exactly what I did in this lab — just at scale and automatically.

| LDAP Concept | InstaSafe AD Sync Equivalent | What it means in practice |
|---|---|---|
| **Base DN** (`dc=lab,dc=instasafe,dc=local`) | **LDAP Base DN in connector config** | This tells InstaSafe where to start looking in the directory tree. If a customer sets the wrong Base DN, InstaSafe won't find any users — it's searching in the wrong folder of the directory. |
| **Bind DN** (`cn=admin,dc=lab,dc=instasafe,dc=local`) | **Service account credentials in AD sync** | InstaSafe needs a service account to log into the customer's AD and read user data. This is the bind DN — the identity InstaSafe uses to authenticate to the directory before it can query anything. |
| **Bind password** (`LabAdmin123!`) | **Service account password stored in InstaSafe** | The password for the service account. If this changes in AD and isn't updated in InstaSafe, every sync fails with error 49. |
| **objectClass: inetOrgPerson** | **User object filter in sync config** | InstaSafe filters the directory to only pull user objects, not computers or printers. In AD this is typically `(objectClass=user)` or `(objectClass=person)`. |
| **Attributes: cn, mail, uid** | **Attribute mapping in InstaSafe** | InstaSafe maps LDAP attributes to its own user fields. `cn` becomes the display name, `mail` becomes the login email, `uid` becomes the username. If the attribute names differ in a customer's AD, the mapping needs to be configured. |
| **ou=People** | **User OU scope in connector** | The connector can be scoped to a specific OU so it only syncs users from that branch, not the entire directory. Useful when a customer wants to sync only a specific department. |
| **ldapwhoami bind test** | **Test connection button in AD sync UI** | Before InstaSafe starts syncing, it does a bind test — exactly like `ldapwhoami`. If the test fails, the sync won't run. This is how you verify credentials and connectivity before going live. |
| **Error 49** | **Authentication failure in sync logs** | When InstaSafe logs an AD sync failure, error 49 means the bind credentials are wrong. A support engineer checks this first before looking at anything else. |

---

## What I Learned

**LDAP is just a structured database with a tree shape.** Before this lab I thought of Active Directory as a black box. After building it from scratch, it's just entries in a tree, each with a unique address (DN) and a set of attributes. Once you see the structure, AD troubleshooting stops being mysterious.

**The bind is the authentication.** Every LDAP authentication is just a bind operation — send a DN and password, get accepted or rejected. There's no session token, no cookie. It's stateless. Understanding this made it clear why error 49 is always the first thing to check when AD sync breaks.

**DN format matters more than people think.** The full DN has to be exactly right — every comma, every `dc=`, every `ou=`. A single typo gives you either error 32 (DN not found) or error 49 (DN found but wrong). I mistyped the base DN once during setup and spent time figuring out why my ldapsearch was returning nothing.

**LDIF is how you bulk-manage directory entries.** In a real AD environment, user provisioning is automated — scripts generate LDIF files or use the AD API to create thousands of users at once. Doing it manually for two users made me appreciate what that automation is doing under the hood.

**InstaSafe's AD sync is just a polished version of exactly this.** The connector config fields — Base DN, Bind DN, password, attribute mapping — are the same parameters I set manually in this lab. Knowing what each one does makes it much easier to help a customer troubleshoot a broken sync.

---

## Final Checklist

| Criteria | Status |
|---|---|
| slapd installed and running | ✅ |
| Base DN configured as `dc=lab,dc=instasafe,dc=local` | ✅ |
| ldapsearch returns base directory structure | ✅ |
| ou=People and ou=Groups created | ✅ |
| alice and bob created with mail attributes | ✅ |
| ldapsearch returns both users — numEntries: 2 | ✅ |
| Email-based user lookup working | ✅ |
| ldapwhoami bind success for alice | ✅ |
| Error 49 captured with wrong password | ✅ |
| Error 49 explanation written | ✅ |
| LDAP → InstaSafe AD sync mapping written | ✅ |

