# 🏹 Lian_Yu Writeup

**Level:** Beginner  
**Platform:** TryHackMe  

---

## 📌 Overview

This room involves enumeration, directory discovery, FTP access, steganography, and privilege escalation to capture user and root flags.

---

## 🔍 Task 1: Web Directory

**Question:** What is the Web Directory you found?

### Approach:

- Performed an Nmap scan:
  - Open ports: `21 (FTP)`, `22 (SSH)`, `80 (HTTP)`, `111`
- Accessed the web server on port 80
- Used Gobuster for directory brute-forcing
- Found directory: `/island`

### Findings:

- Viewing page source revealed FTP username: `vigilante`
- Continued enumeration and found another directory: `/2100`
- Found a file with `.ticket` extension

### ✅ Answer: 2100

---

## 🔍 Task 2: File Name

**Question:** What is the file name you found?

### Approach:

- Used Gobuster with extension flag:-x ticket
- Discovered file:

### ✅ Answer: green_arrow.ticket


---

## 🔍 Task 3: FTP Password

**Question:** What is the FTP password?

### Approach:

- Retrieved file content using curl
- Found encoded string: RTy8yhBQdscX

- Decoded (Base58/Base57-like encoding)



### ✅ Answer: !#th3h00d 

---

## 🔍 Task 4: SSH Password File

**Question:** What is the file name with SSH password?

### Approach:

- Logged into FTP:
    - Username: vigilante
    - Password: !#th3h00d

- Navigated directories and found another user: `slade`
- Downloaded all files:
``` mget .* * ```

- Identified `aa.jpg` as a steganographic file
- Used `stegseek` with `rockyou.txt`:
- Extracted: `aa.jpg.out`

### Extracted Files:
- `password.txt`
- `shado`

- Found SSH password inside `shado`

### ✅ Answer:shado

---

## 🔍 Task 5: User Flag

**Question:** user.txt ?

### Approach:

- SSH login:
    - Username: slade
    - Password: M3tahuman
    
- Located and read `user.txt`

### ✅ Answer:  THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}


---

## 🔍 Task 6: Root Flag

**Question:** root.txt ?

### Approach:

- Checked sudo permissions:     
    - sudo -l
- Found permission to run:
    - /usr/bin/pkexec
- Escalated privileges:
    - sudo /usr/bin/pkexec /bin/sh
- Accessed root and read `root.txt`

### ✅ Answer:THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}
