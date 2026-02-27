# botpruebas
pruebas bot
$ErrorActionPreference = "Stop"
Set-Location C:\Monitor

# venv con Python actual (3.13)
python -m venv .venv

# actualizar pip
.\.venv\Scripts\python.exe -m pip install --upgrade pip setuptools wheel

# limpiar paquetes problemáticos
.\.venv\Scripts\pip.exe uninstall -y pysnmp pyasn1 pysnmp-mibs

# instalar versión compatible con Python 3.13
.\.venv\Scripts\pip.exe install pyasn1 requests python-telegram-bot==13.15 WMI pywin32 mss Pillow pymodbus pyodbc pandas

# instalar pysnmp LEAN sin MIB builder legacy
.\.venv\Scripts\pip.exe install pysnmp-lextudio

# verificación rápida
.\.venv\Scripts\python.exe -c "import importlib; print('importlib.util=',hasattr(importlib,'util')); import requests,telegram,pyasn1,wmi,pythoncom,mss,PIL,pymodbus,pyodbc,pandas; from pysnmp.hlapi import SnmpEngine; print('OK_IMPORTS')"

# arrancar bot
.\.venv\Scripts\python.exe .\bot_main.py
