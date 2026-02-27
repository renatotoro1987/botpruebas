@'
@echo off
cd /d C:\Monitor
C:\Monitor\.venv\Scripts\python.exe C:\Monitor\Buzys.py >> C:\Monitor\kpi_daily.log 2>&1
'@ | Set-Content -Path C:\Monitor\run_kpi_daily.bat -Encoding ascii



schtasks /Create /TN "Bot\KPI_Daily_Busy" /SC DAILY /ST 00:15 /TR "C:\Monitor\run_kpi_daily.bat" /RU "SYSTEM" /RL HIGHEST /F


schtasks /Run /TN "Bot\KPI_Daily_Busy"



schtasks /Query /TN "Bot\KPI_Daily_Busy" /V /FO LIST
Get-Content C:\Monitor\kpi_daily.log -Tail 40
