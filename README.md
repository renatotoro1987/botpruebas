import requests
import re

BASE_URL = "http://10.100.105.32"

session = requests.Session()

headers = {
    "User-Agent": "Mozilla/5.0",
}

def iniciar_sesion():
    # Paso 1: entrar al index (esto genera cookies)
    r = session.get(BASE_URL + "/index.html", headers=headers, timeout=10)
    r.raise_for_status()
    print("Index cargado OK")

def obtener_sesion():
    payload = '<ioReq op="connect"/>'

    r = session.post(BASE_URL + "/ObjectIO",
                     headers={"Content-Type": "text/xml"},
                     data=payload,
                     timeout=10)

    print("Respuesta connect:\n", r.text)

    match = re.search(r'id="([a-z0-9]+)"', r.text)
    if match:
        return match.group(1)
    return None

if __name__ == "__main__":
    iniciar_sesion()
    session_id = obtener_sesion()
    print("Session ID:", session_id)
