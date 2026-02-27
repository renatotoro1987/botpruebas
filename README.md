# botpruebas
pruebas bot


$ErrorActionPreference = "Stop"
Set-Location C:\Monitor

# 1) Instalar Python 3.11 (sin winget)
$installer = "$env:TEMP\python-3.11.9-amd64.exe"
Invoke-WebRequest -Uri "https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe" -OutFile $installer
Start-Process -FilePath $installer -ArgumentList "/quiet InstallAllUsers=0 PrependPath=1 Include_test=0 Include_launcher=1" -Wait

$PY311 = "$env:LocalAppData\Programs\Python\Python311\python.exe"
if (!(Test-Path $PY311)) { throw "No se instaló Python 3.11 correctamente." }

# 2) Limpiar posibles conflictos locales de importlib
Get-ChildItem C:\Monitor -Recurse -File | Where-Object { $_.Name -like "importlib*" } | Select-Object FullName

# 3) requirements
@'
python-telegram-bot==13.15
requests==2.32.5
pysnmp==4.4.12
pyasn1==0.4.8
pysnmp-mibs==0.1.6
WMI==1.5.1
pywin32==311
mss==10.1.0
Pillow==12.1.1
pymodbus==3.8.4
pyodbc==5.3.0
pandas==2.2.3
'@ | Set-Content -Path C:\Monitor\requirements.txt -Encoding ascii

# 4) Venv limpio con 3.11
if (Test-Path C:\Monitor\.venv) { Remove-Item C:\Monitor\.venv -Recurse -Force }
& $PY311 -m venv C:\Monitor\.venv

# 5) Instalar dependencias
& C:\Monitor\.venv\Scripts\python.exe -m pip install --upgrade pip setuptools wheel
& C:\Monitor\.venv\Scripts\pip.exe install -r C:\Monitor\requirements.txt

# 6) Verificación
& C:\Monitor\.venv\Scripts\python.exe -c "import importlib; print('importlib_file=',importlib.__file__); print('has_util=',hasattr(importlib,'util')); import telegram,requests,pysnmp,pyasn1,wmi,pythoncom,mss,PIL,pymodbus,pyodbc,pandas; print('OK_IMPORTS')"

# 7) Ejecutar bot
& C:\Monitor\.venv\Scripts\python.exe C:\Monitor\bot_main.py

