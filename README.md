import requests
import re

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

CHANNEL_ID = "08d29ed9bc1528"

PAYLOAD = """<ioReq op="subscribeEx" id="{id}"><address type="te"><te>0</te></address><object name="Slot4:performParamReading.1"/></ioReq>""".format(
    id=CHANNEL_ID
)

def main():
    try:
        r = requests.post(URL, headers=HEADERS, data=PAYLOAD, timeout=15)
        r.raise_for_status()

        print("HTTP:", r.status_code)
        print("Respuesta:\n")
        print(r.text)

        match = re.search(r'value="(-?\d+)"', r.text)
        if match:
            valor = int(match.group(1)) / 100
            print("\nRSL: {} dBm".format(valor))
        else:
            print("\nNo se encontró un value numérico.")

    except Exception as e:
        print("Error:", e)

if __name__ == "__main__":
    main()
