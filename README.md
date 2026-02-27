Set-Location C:\Monitor

@'
@echo off
cd /d C:\Monitor
C:\Monitor\.venv\Scripts\python.exe C:\Monitor\mirror_to_vps.py
'@ | Set-Content -Path C:\Monitor\run_mirror.bat -Encoding ascii

schtasks /Create /TN "Bot\MirrorToVPS" /SC MINUTE /MO 5 /TR "C:\Monitor\run_mirror.bat" /RU "SYSTEM" /RL HIGHEST /F

schtasks /Run /TN "Bot\MirrorToVPS"
schtasks /Query /TN "Bot\MirrorToVPS" /V /FO LIST



schtasks /Create /TN "Bot\MirrorToVPS_Startup" /SC ONSTART /TR "C:\Monitor\run_mirror.bat" /RU "SYSTEM" /RL HIGHEST /F


Get-Content C:\Monitor\mirror_to_vps.log -Tail 30


