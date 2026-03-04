# Mail a ZKTeco — Achat SDK TA PUSH + Documentation technique

**A :** sales@zkteco.com
**Objet :** Purchase inquiry — TA PUSH SDK license + technical documentation for Horus E1-FP

---

Hi,

We are developing a SaaS attendance platform and have selected the **ZKTeco Horus E1-FP** as our biometric device.

We are purchasing the hardware separately and need to buy the **TA PUSH SDK** license directly from ZKTeco.

## What we need

1. **TA PUSH SDK license** — to enable the Horus E1-FP to push attendance data directly to our cloud server (Vercel/Next.js) via HTTP over 4G
2. **Complete SDK documentation** (see section below)

## Questions about the SDK

1. **Price:** What is the cost of the TA PUSH SDK license? Is it a one-time fee or annual subscription?
2. **Per-device or universal?** Does the license cover one device or multiple devices?
3. **Compatibility:** Is the TA PUSH SDK compatible with the Horus E1-FP (FacePro series, 4G)?
4. **Architecture:** Does the device push directly to our server (HTTP POST), or does it require a ZKTeco intermediary server (UTimeMaster, ADMS, etc.)?

## Documentation request — before purchase

Before we purchase the SDK, we need to review the technical documentation to confirm it meets our requirements. Could you please share:

1. **TA PUSH SDK Developer Guide** — the full technical documentation describing:
   - The HTTP protocol between the device and our server
   - How to configure the push/callback URL on the device
   - The format of push requests (attendance data, heartbeat, device status)
   - How to send commands back to the device (add/delete users, enrollment)
   - Authentication mechanism between device and server

2. **Latest version available** — what is the current version of the TA PUSH SDK? We previously received a PDF titled "UTimeMaster API User Manual (2019)", but this describes the UTimeMaster software, not the TA PUSH SDK. We need the documentation specific to the Push SDK.

3. **Changelog / release notes** — is there a list of versions and what changed between them? We want to make sure we get the latest features and fixes.

4. **Sample request/response** — a concrete example of:
   - An attendance push from the device to our server (JSON format)
   - A command sent from our server to the device (e.g., add user)
   - A heartbeat/status message from the device

5. **Integration guide or quick start** — any guide that shows how to set up the Push SDK from scratch with a custom server

## Our architecture

```
Horus E1-FP (4G SIM) --HTTP POST--> Our cloud server (Vercel/Next.js API route)
                                          |
                                          v
                                     PostgreSQL database (Supabase)
```

No UTimeMaster. No ADMS. No local PC. Just the device pushing directly to our server via 4G.

We need to confirm this architecture is supported before purchasing the SDK.

We are ready to proceed with the purchase as soon as we validate the documentation.

Thank you for your time.

Best regards
