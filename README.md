






# n8n_full_report.py
# -- coding: utf-8 --

import platform
import subprocess
import datetime
import os
import json
import re
import requests

try:
    import wmi
except ImportError:
    wmi = None

try:
    import pythoncom
except ImportError:
    pythoncom = None

from pysnmp.hlapi import (
    SnmpEngine, CommunityData, UdpTransportTarget,
    ContextData, ObjectType, ObjectIdentity, getCmd
)

from config import (
    SOURCE_IP, SITES, RPTS,
    NTI_IP, OID_AC, OID_DC, OID_TEMP,
    COMMUNITY, COMMUNITY_RPT, OID_MOTOTRBO_STATUS,
    WINDOWS_HOST, WINDOWS_USER,
    WINDOWS_PASSWORD, USO_DIARIO_PROMEDIO_GB,
    N8N_WEBHOOK_URL_REPORTE_DIARIO,
    PING_COUNT, PING_TIMEOUT_MS, PING_MIN_SUCCESS
)


def save_payload_to_file(payload, folder="logs"):
    os.makedirs(folder, exist_ok=True)
    ts = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    path = os.path.join(folder, f"full_report_{ts}.json")
    with open(path, "w", encoding="utf-8") as f:
        json.dump(payload, f, ensure_ascii=False, indent=2)
    return path


def _extract_received_count(output: str) -> int:
    text = output or ""
    m = re.search(r"(?:received|recibidos)\s*=\s*(\d+)", text, flags=re.IGNORECASE)
    if m:
        return int(m.group(1))
    m = re.search(r"(\d+)\s+(?:packets\s+)?received", text, flags=re.IGNORECASE)
    if m:
        return int(m.group(1))
    return -1


def ping_host(host, count=None, timeout_ms=None, min_success=None):
    count = int(count if count is not None else PING_COUNT)
    timeout_ms = int(timeout_ms if timeout_ms is not None else PING_TIMEOUT_MS)
    min_success = int(min_success if min_success is not None else PING_MIN_SUCCESS)

    count = max(1, count)
    timeout_ms = max(100, timeout_ms)
    min_success = max(1, min_success)

    if platform.system().lower() == "windows":
        cmd = ["ping", "-n", str(count), "-w", str(timeout_ms), "-S", SOURCE_IP, host]
    else:
        timeout_sec = max(1, int((timeout_ms + 999) / 1000))
        cmd = ["ping", "-c", str(count), "-W", str(timeout_sec), "-I", SOURCE_IP, host]

    try:
        proc = subprocess.run(
            cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            encoding="utf-8",
            errors="ignore",
        )
        received = _extract_received_count(proc.stdout)
        if received >= 0:
            return received >= min_success
        return proc.returncode == 0
    except Exception:
        return False


def snmp_get(ip, oid):
    iterator = getCmd(
        SnmpEngine(),
        CommunityData(COMMUNITY),
        UdpTransportTarget((ip, 161), timeout=2, retries=1),
        ContextData(),
        ObjectType(ObjectIdentity(oid))
    )
    try:
        error_indication, error_status, error_index, var_binds = next(iterator)
        if error_indication or error_status:
            return None
        for var_bind in var_binds:
            return str(var_bind[1])
    except Exception:
        return None


def snmp_get_with_community(ip, oid, community, version=1):
    iterator = getCmd(
        SnmpEngine(),
        CommunityData(community, mpModel=0 if version == 1 else 1),
        UdpTransportTarget((ip, 161), timeout=2, retries=1),
        ContextData(),
        ObjectType(ObjectIdentity(oid))
    )
    try:
        error_indication, error_status, error_index, var_binds = next(iterator)
        if error_indication or error_status:
            return None
        for var_bind in var_binds:
            return str(var_bind[1])
    except Exception:
        return None


def rpt_is_up(ip):
    """
    Repetidor UP = responde ping y entrega estado SNMP MOTOTRBO válido.
    """
    if not ping_host(ip):
        return False

    status = snmp_get_with_community(ip, OID_MOTOTRBO_STATUS, COMMUNITY_RPT, version=1)
    if not status:
        return False

    return "MOTOTRBO" in str(status).upper()


def check_windows_storage():
    if wmi is None:
        return [{
            "unidad": "N/A",
            "espacio_libre_gb": None,
            "tamano_total_gb": None,
            "dias_restantes": None,
            "estado": "Sin conexión: librería wmi no instalada"
        }]

    if pythoncom is not None:
        pythoncom.CoInitialize()

    try:
        conn = wmi.WMI(
            computer=WINDOWS_HOST,
            user=WINDOWS_USER,
            password=WINDOWS_PASSWORD
        )

        storage = []
        for disk in conn.Win32_LogicalDisk(DriveType=3):
            size = int(disk.Size or 0)
            free = int(disk.FreeSpace or 0)

            free_gb = free / (1024**3) if free else 0
            size_gb = size / (1024**3) if size else 0

            dias = None
            if USO_DIARIO_PROMEDIO_GB and USO_DIARIO_PROMEDIO_GB > 0:
                dias = int(free_gb / USO_DIARIO_PROMEDIO_GB)

            storage.append({
                "unidad": disk.Caption,
                "espacio_libre_gb": round(free_gb, 2),
                "tamano_total_gb": round(size_gb, 2),
                "dias_restantes": dias,
                "DD": dias,
                "estado": "OK"
            })

        if not storage:
            return [{
                "unidad": "N/A",
                "espacio_libre_gb": None,
                "tamano_total_gb": None,
                "dias_restantes": None,
                "estado": "Sin discos detectados o error de permisos WMI"
            }]

        return storage

    except Exception as e:
        return [{
            "unidad": "N/A",
            "espacio_libre_gb": None,
            "tamano_total_gb": None,
            "dias_restantes": None,
            "estado": f"Sin conexión: {type(e).__name__}: {e}"
        }]

    finally:
        if pythoncom is not None:
            try:
                pythoncom.CoUninitialize()
            except Exception:
                pass


def collect_full_report():
    payload = {
        "success": True,
        "timestamp": datetime.datetime.now().isoformat(),
        "sites": {},
        "repeaters": {},
        "sensors": {},
        "server_storage": check_windows_storage()
    }

    temp_repeater_status = {}

    for site_name, rpts_dict in RPTS.items():
        site_states = {}
        for rpt_name, rpt_ip in rpts_dict.items():
            is_up = rpt_is_up(rpt_ip)
            site_states[rpt_name] = is_up
        payload["repeaters"][site_name] = site_states
        temp_repeater_status[site_name] = site_states

    for site_name, main_ip in SITES.items():
        if site_name in temp_repeater_status:
            rpts_del_sitio = temp_repeater_status[site_name]
            valores = rpts_del_sitio.values()
            site_status = any(valores)
        else:
            site_status = ping_host(main_ip)

        payload["sites"][site_name] = site_status

    ac = snmp_get(NTI_IP, OID_AC)
    dc = snmp_get(NTI_IP, OID_DC)
    temp = snmp_get(NTI_IP, OID_TEMP)

    payload["sensors"] = {
        "voltaje_ac": float(ac) / 10 if ac else "Sin conexión",
        "voltaje_dc": float(dc) / 10 if dc else "Sin conexión",
        "temperatura": float(temp) / 10 if temp else "Sin conexión"
    }

    cmss_principal_ip = "10.109.85.1"
    cmss_secundario_ip = "10.109.85.65"

    payload["cmss_controllers"] = {
        "CMSS_principal_trunking_controller": ping_host(cmss_principal_ip),
        "CMSS_secundario_trunking_controller": ping_host(cmss_secundario_ip)
    }

    return payload


def enviar_reporte_completo_n8n():
    try:
        payload = collect_full_report()
        saved_path = save_payload_to_file(payload)
        print(f"JSON guardado en: {saved_path}")

        resp = requests.post(N8N_WEBHOOK_URL_REPORTE_DIARIO, json=payload, timeout=20)

        if 200 <= resp.status_code < 300:
            try:
                response_data = resp.json()
                if response_data.get("success") == "true":
                    return "Reporte completo enviado por correo a la lista de difusión."
                return f"n8n respondió, pero success no es 'true'. Respuesta: {response_data}"
            except ValueError:
                return f"n8n respondió código {resp.status_code} pero no devolvió un JSON válido."

        return f"Error HTTP n8n: Código {resp.status_code}"

    except Exception as e:
        return f"Error enviando a n8n: {e}"


if __name__ == "__main__":
    print("Probando generación de reporte...")
    resultado = collect_full_report()
    ruta = save_payload_to_file(resultado)
    print(f"JSON guardado en: {ruta}")
    print(json.dumps(resultado, ensure_ascii=False, indent=4))
