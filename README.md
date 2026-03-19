import requests

URL = "http://10.100.105.32/ObjectIO"

HEADERS = {
    "Accept": "*/*",
    "Accept-Language": "es-ES,es;q=0.9",
    "Connection": "keep-alive",
    "Content-Type": "application/xml;charset=UTF-8",
    "Origin": "http://10.100.105.32",
    "Referer": "http://10.100.105.32/index.htm",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36",
}

PAYLOAD = """<ioReq op="subscribeEx" id="08d29ed9bc1528"><address type="te"><te>0</te></address><object name="terminal:mfgDetailsInfoUnitType.1"/><object name="terminal:sysDetailsInfoRadioName.0"/><object name="terminal:sysDetailsInfoSiteName.0"/><object name="terminal:sysDetailsInfoContactDetails.0"/><object name="terminal:ipConfigAdEntAddress.1"/><object name="terminal:swDetailsVersion.1"/></ioReq>"""

def main():
    try:
        r = requests.post(URL, headers=HEADERS, data=PAYLOAD, timeout=15)
        r.raise_for_status()

        print("HTTP:", r.status_code)
        print("Respuesta:\n")
        print(r.text)

    except Exception as e:
        print("Error:", e)

if __name__ == "__main__":
    main()




PS C:\Aviat> python .\p4.py
HTTP: 200
Respuesta:

<?xml version="1.0" encoding="UTF-8"?><ioRes id="08d29ed9bc1528"><res op="subscribeEx"><address type="full"><ne>10.100.1
05.32</ne><te>0</te></address><object name="terminal:mfgDetailsInfoUnitType.1" value="NCC" dataType="string"><valueType>
<value min="0" max="32"/></valueType></object><object name="terminal:sysDetailsInfoRadioName.0" value="Unnamed Radio. IN
Uv3" dataType="string"><valueType><value min="0" max="32"/></valueType></object><object name="terminal:sysDetailsInfoSit
eName.0" value="Site name not defined" dataType="string"><valueType><value min="0" max="32"/></valueType></object><objec
t name="terminal:sysDetailsInfoContactDetails.0" value="" dataType="string"><valueType><value min="0" max="64"/></valueT
ype></object><object name="terminal:ipConfigAdEntAddress.1" value="10.100.105.32" dataType="ipAddress"/><object name="te
rminal:swDetailsVersion.1" value="08.17.03" dataType="string"><valueType><value min="0" max="64"/></valueType></object><
/res></ioRes>
PS C:\Aviat>



