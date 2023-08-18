---
layout: post
category: cheatsheets
---

## Malleable C2:

1. `Memory Permissions & Cleanup`
2. `BOF Memory Allocations`
3. SPawnTo to Notepad
4. Smb PIPE NAMES
5. Amsi Disable CRTO
6. sleep_mask

## Kits: 
1. `Process Inject Kit`
2. `Event Tracing For Windows` in aggressor script.
3. `Inline (.NET) Execution` [--amsi --etw --appdomain SharedDomain --pipe dotnet-diagnostic-1337]
4. `Thread Stack Spoofing`
5. Build `Sleep mask kit`, follow along with module.
6. Mimikatz kit

## OPPSEC:
1. Use SpawnTo See below then ppid spoof
2. argue
3. Command line exclusions - `Process Creastions from PSExec & Wmi`

EDR Evasion:
1. `Process Mitigation Policy` - blockdlls
2. list_process_callbacks 
3. zero_process_callback 

### IF REDIRECTOR BREAKS: sudo update-ca-certificates

- Powershell SpawnTo 

`spawnto x64 %windir%\sysnative\msiexec.exe`

- ldap SpawnTo

`spawnto x64 %windir%\sysnative\gpresult.exe`

- dllhost SpawnTo

`spawnto x64 %windir%\sysnative\dllhost.exe /Processid:{11111111-2222-3333-4444-555555555555}`

- Mimikatz SpawnTo 

`spawnto x64 c:\windows\system32\mrt.exe`
