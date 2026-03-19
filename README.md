import requests
import re
import uuid

URL = "http://10.100.105.32/ObjectIO"

HEADERS = {
    "Content-Type": "application/xml;charset=UTF-8",
}

def generar_id():
    # Genera un ID tipo session válido
    return uuid.uuid4().hex[:16]

def obtener_rsl():
    channel_id = generar_id()

    payload = f"""
    <ioReq op="subscribeEx" id="{channel_id}">
        <address type="te"><te>0</te></address>
        <object name="Slot4:performParamReading.1"/>
    </ioReq>
    """

    r = requests.post(URL, headers=HEADERS, data=payload, timeout=10)
    r.raise_for_status()

    match = re.search(r'value="(-?\d+)"', r.text)

    if match:
        return int(match.group(1)) / 100
    return None

if __name__ == "__main__":
    try:
        rsl = obtener_rsl()
        print(rsl if rsl is not None else 0)
    except:
        print(0)
