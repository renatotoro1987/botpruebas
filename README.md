1690A670-A132-11EF-B866-BBC00315E515


schtasks /Query /TN "Bot\BotMain" /V /FO LIST
schtasks /Query /TN "Bot\MirrorToVPS" /V /FO LIST



@'
@echo off
cd /d C:\Monitor
C:\Monitor\.venv\Scripts\python.exe C:\Monitor\bot_main.py >> C:\Monitor\bot_main.log 2>&1
'@ | Set-Content -Path C:\Monitor\run_bot_main.bat -Encoding ascii

schtasks /Create /TN "Bot\BotMain" /SC ONSTART /DELAY 0000:30 /TR "C:\Monitor\run_bot_main.bat" /RU "SYSTEM" /RL HIGHEST /F
schtasks /Run /TN "Bot\BotMain"




Get-Content C:\Monitor\bot_main.log -Tail 80
