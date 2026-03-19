import requests
import re

URL = "http://10.100.105.32/ObjectIO"

headers = {
    "User-Agent": "Mozilla/5.0",
    "Content-Type": "text/xml"
}

def obtener_sesion():
    payload = '<ioReq op="connect" id="1234567890abc"/>'

    r = requests.post(URL, headers=headers, data=payload, timeout=10)
    r.raise_for_status()

    texto = r.text
    print("Connect response:\n{}".format(texto))

    match = re.search(r'id="([a-z0-9]+)"', texto)
    if match:
        return match.group(1)
    else:
        return None

def leer_rsl(session_id):
    payload = """<?xml version="1.0" encoding="UTF-8"?>
<ioReq op="get" id="{}">
    <object name="Slot1:performParamReading.1"/>
</ioReq>
""".format(session_id)

    r = requests.post(URL, headers=headers, data=payload, timeout=10)
    r.raise_for_status()

    texto = r.text
    print("\nRespuesta RSL:\n{}".format(texto))

    match = re.search(r'value="(-?\d+)"', texto)
    if match:
        valor = int(match.group(1)) / 100
        print("\nRSL: {} dBm".format(valor))
    else:
        print("\nNo encontrado")

if __name__ == "__main__":
    try:
        session_id = obtener_sesion()
        if session_id:
            print("\nSession ID: {}".format(session_id))
            leer_rsl(session_id)
        else:
            print("No se pudo obtener sesión")
    except Exception as e:
        print("Error: {}".format(e))
