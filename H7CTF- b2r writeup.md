
This writeup details the complete compromise of the H7Corp machine, a multi-stage challenge that required chaining several vulnerabilities: information disclosure, broken access control, insecure deserialization, and a command injection in a SUID binary to pivot users and exploit a privileged gRPC service.


Our initial `nmap` scan revealed ports 22 (SSH) and 80 (HTTP). The most critical finding was from the `http-git` nmap script, which detected an exposed `/.git` directory.


 Information Disclosure (Leaked `.git` Repository)** We used `git-dumper` to download the entire contents of the exposed repository. This gave us the application's source code, most importantly a `proto` directory containing Protocol Buffer (gRPC) API definitions:

- `auth.proto`
    
- `user.proto`
    
- `backup.proto`
    
- `monitoring.proto`
    
- `internal.proto` (This one, defining an `AdminService` with `GetRootShell`, was our primary suspect for privilege escalation).
The presence of gRPC definitions prompted a full port scan (`nmap -p-`) to find the service. This revealed a non-standard port: **`50051/tcp`**.

**gRPC Enumeration (Reflection Bypass)** We used `grpcurl` to probe this port.

- **Attempt 1:** `grpcurl -plaintext 10.10.91.168:50051 list`
    
- **Result:** `Failed to list services: server does not support the reflection API`.
    
- **Bypass:** We bypassed this by using the `.proto` files we had already leaked. We tested the `AdminService` from `internal.proto`, but it failed (`Method not found!`), indicating it was not the service running here.
    
- **Success:** We then tested the `monitoring.proto` file:
-grpcurl -plaintext -proto monitoring.proto -import-path . -d '{}' 10.10.91.168:50051 cpr.monitoring.MetricsService.GetServiceHealth

This worked and, in its response, **leaked the names of all _other_ active services**:

- `cpr.auth.AuthService`
    
- `cpr.user.UserService`
    
- `cpr.backup.BackupService`




- **Register a User:** We used `auth.proto` to register a new user, "akash2," which returned a `userId`.
    
    - **Command:** `grpcurl -plaintext -proto auth.proto ... RegisterUser`
        
    - **Result:** `{"success": true, "message": "User registered successfully", "userId": "usr_6434e7208904"}`
        
- **Find Broken Access Control (BAC):** We tried to use the `BackupService` with our new ID but were denied: `{"message": "Admin privileges required"}`. This meant we needed admin rights. We then probed the `UserService`, finding the `UpdateProfile` function.
    
    - **Command:** `grpcurl -plaintext -proto user.proto ... -d '{"user_id": "usr_6434e7208904", ... "is_admin": true}' ... UpdateProfile`
        
    - **Result:** `{"success": true, "message": "Profile updated successfully"}`.
        
    - **Vulnerability:** The API insecurely allowed us to set our own admin status. We were now an application-level admin.
![[Pasted image 20251115181842.png]]
![[Pasted image 20251115181901.png]]
        
- **Find Insecure Deserialization (RCE):** Now as an admin, we re-tried the `BackupService`, specifically the `RestoreBackup` function. We sent junk data to see how it would fail.
    
    - **Result:** `{"message": "Restore failed: unpickling stack underflow"}`.
        
    - **Vulnerability:** This error message was the key. It confirmed the server was using Python's `pickle` module, which is notoriously vulnerable to Remote Code Execution (RCE).
    - ![[Pasted image 20251115182104.png]]
    - 
    - 
![[Pasted image 20251115181438.png]]

![[Pasted image 20251115181944.png]]
        
- **Get Shell:** We crafted a Python script to create a malicious `pickle` payload that would execute a reverse shell. We sent this as the `backup_data` and successfully received a reverse shell as the `svc-runtime` user.
- 
**Local Enumeration and User Flag** On the server as `svc-runtime`, we read the application source code at `/opt/cpr/app/server.py`. This file contained the hardcoded user flag: **`FLAG_1 = "H7CTF{d1d_y0u_l3arn_ab0ut_pr0t0buf_inject10ns}"`**.
![[Pasted image 20251115182207.png]]
Our goal was now root. After checking common vectors (`sudo -l`, `crontab`, `getcap`), we found two critical clues:

1. **Root Service:** A Unix socket existed at `/run/cpr-admin.sock`.
    
2. **Permissions:** This socket was owned by `root` but group-accessible by **`platform-admins`**.

    

To get root, we first had to become a member of the `platform-admins` group. `ls -la /home` showed a user named `abu` belonged to this group.
**Vulnerability 3: SUID Command Injection** We searched for a way to become `abu`. Also i found there  was a password file stored in the shell and tried to see through it but all i got was the hash of the password , which then i tried to get it in hashcracker which failed 

The real vector was a SUID binary.

- **Vector:** `find / -user abu` revealed that `abu` owned a SUID binary: `/usr/local/bin/cpr-backup`. This meant the program would run with `abu`'s permissions.
    
- **Vulnerability:** `strings` on the binary revealed it was vulnerable to command injection: `cd %s && tar -czf %s/%s.tar.gz . 2>/dev/null`.
- ![[Pasted image 20251115182421.png]]
After several failed attempts due to shell syntax errors (`Bad fd number`), we crafted the correct payload. This payload uses a semicolon to start a new command, double quotes to correctly pass the reverse shell string to `bash -c`, and a `#` to comment out the rest of the original command
![[Pasted image 20251115183130.png]]
![[Pasted image 20251115183136.png]]

The SUID binary (running as `abu`) executed our payload, and we received a new shell as the `abu` user.
**Insecure Privileged Service (Root)** Now as `abu`, we were in the `platform-admins` group and had permission to access the root socket.


1**File Transfer:** We transferred the `grpcurl` binary and the `internal.proto` file to the `/tmp` directory.
 ![[Pasted image 20251115183742.png]]
 ![[Pasted image 20251115183759.png]]
 ![[Pasted image 20251115183944.png]]
    
. **Exploit:** We used `grpcurl` to connect to the Unix socket and call the `AdminService.ExecuteCommand` function. We fixed a final `Bad fd number` error by using double quotes for our payload.
![[Pasted image 20251115184047.png]]


- **Success:** The `AdminService` (running as root) executed our payload. We received a connection on port 4447, and we were **root**.
    
- **Root Flag:** The file `/root/root.txt` did not exist. The final flag was found by  find / -name "flag.txt" 2>/dev/null  like i tried for root.txt but didnt get anything first 







![[Pasted image 20251115184813.png]]

![[Pasted image 20251108200239.png]]



THE FINAL PIC OF WHAT IT LOOKED LIKE




![[Pasted image 20251115184029.png]]


I tried to cover up what all i did to pwn this hope this helps!