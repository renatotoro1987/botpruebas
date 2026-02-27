Set-Location C:\Monitor
.\.venv\Scripts\python.exe -m pip install --upgrade "setuptools==80.9.0" "tzlocal==2.1"
.\.venv\Scripts\python.exe -c "import pkg_resources; print('OK_PKG_RESOURCES')"
.\.venv\Scripts\python.exe .\bot_main.py
