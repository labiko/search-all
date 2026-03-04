# Reponse a ZKTeco — Clarification TA PUSH SDK avant paiement

**A :** [contact ZKTeco]
**Objet :** Re: Horus E1-FP — Important questions about TA PUSH SDK before payment

---

Hi,

Thank you for the quotation. We are ready to proceed with the order ($723 via UPS), but before we pay, we need to clarify a few important points about the TA PUSH SDK.

## About the documentation received

We reviewed the PDF you shared ("UTimeMaster API User Manual - 2019-05-17"). This document describes the **UTimeMaster software API** (a server application), not the TA PUSH SDK protocol itself.

The UTimeMaster API only provides:
- GET endpoints to poll attendance data (no real-time push)
- Employee CRUD on the UTimeMaster server (not directly on the device)
- No biometric enrollment API
- No webhook / callback URL mechanism

This does not match what we need. We need to understand how the **TA PUSH SDK** works specifically.

## Questions we need answered before payment

**1. Does the TA PUSH SDK require a UTimeMaster server?**
We do NOT want to run a UTimeMaster server. We want the Horus E1-FP to push data **directly to our own cloud server** (Vercel/Next.js) via HTTP. Is this possible with the TA PUSH SDK?

**2. How does the push mechanism work?**
When an employee clocks in, does the Horus E1-FP send an HTTP POST directly to a URL we configure? What is the format of the data (JSON)? Can you share a sample request/response?

**3. Can we send commands to the device remotely?**
Can we add/delete users on the Horus E1-FP remotely via HTTP from our server? Or does this require UTimeMaster?

**4. Remote biometric enrollment?**
Can we trigger face/fingerprint enrollment on the device remotely via the TA PUSH SDK?

**5. Can you share the actual TA PUSH SDK documentation?**
We need the technical documentation that describes:
- The HTTP protocol between the device and our server
- How to configure the callback/push URL on the device
- The format of push requests (attendance data, heartbeat, etc.)
- How to send commands back to the device

**6. Architecture diagram?**
Could you share a simple diagram showing the data flow:
```
Horus E1-FP (4G) → ??? → Our cloud server
```
Does it go: Device → Our server directly?
Or: Device → ZKTeco cloud → Our server?
Or: Device → UTimeMaster server → Our server?

## What we want

```
Horus E1-FP (4G SIM) --HTTP POST--> Our Vercel server (Next.js API route)
                                          |
                                          v
                                     Supabase (database)
```

No UTimeMaster. No ZKTeco cloud. No local PC. Just the device pushing directly to our server via 4G.

**Is this possible with the TA PUSH SDK?** If yes, please share the correct documentation. If not, please let us know what the alternatives are.

We are eager to place the order as soon as we confirm the SDK covers our needs.

Best regards
