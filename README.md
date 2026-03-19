import requests
import re

URL = "http://10.100.105.32/ObjectIO"

HEADERS = {
    "Content-Type": "application/xml;charset=UTF-8",
}

CHANNEL_ID = "08d29ed9bc1528"

def get_rsl():
    payload = f"""
    <ioReq op="subscribeEx" id="{CHANNEL_ID}">
        <address type="te"><te>0</te></address>
        <object name="Slot4:performParamReading.1"/>
    </ioReq>
    """

    r = requests.post(URL, headers=HEADERS, data=payload, timeout=10)

    if "Channel id" in r.text:
        return None

    match = re.search(r'value="(-?\d+)"', r.text)
    if match:
        return int(match.group(1)) / 100

    return None

if __name__ == "__main__":
    try:
        rsl = get_rsl()
        print(rsl if rsl is not None else 0)
    except:
        print(0)
