"""
System Advisor (SA) Data Collector Service
==========================================
Polls System Advisor data from the network management system,
collects system metrics and status information,
and stores the data in a CSV file on the desktop.

Requirements:
  Install Net-SNMP (snmpget.exe must be available)

Usage:
  python SA_Collector.py

Runs indefinitely and updates CSV every poll interval until Ctrl+C.
"""

import os
import subprocess
import time
import csv
from datetime import datetime
from pathlib import Path


# ===================================================================
#  Configuration
# ===================================================================

SA_HOST = "10.109.85.5"
SA_SNMP_PORT = 161
SNMP_USER = "public"
SNMP_VERSION = "2c"
NMS_IP = "10.109.85.17"
POLL_INTERVAL = 60
CSV_FILENAME = "Historical_SA_Data.csv"

DESKTOP_PATH = Path.home() / "Desktop" / CSV_FILENAME


# ===================================================================
#  OID Mappings
# ===================================================================

OID_SYSNAME = "1.3.6.1.2.1.1.5.0"
OID_SYSDESCR = "1.3.6.1.2.1.1.1.0"
OID_SYSUPTIME = "1.3.6.1.2.1.1.3.0"
OID_SYSCONTACT = "1.3.6.1.2.1.1.4.0"
OID_SYSLOCATION = "1.3.6.1.2.1.1.6.0"
OID_CPUUSAGE = "1.3.6.1.4.1.161.19.3.2.2.1.1.0"
OID_MEMORYUSAGE = "1.3.6.1.4.1.161.19.3.2.2.1.2.0"
OID_DISKUSAGE = "1.3.6.1.4.1.161.19.3.2.2.1.3.0"
OID_INTERFACECNT = "1.3.6.1.2.1.2.1.0"

OID_MAP = {
    "System_Name": OID_SYSNAME,
    "System_Description": OID_SYSDESCR,
    "System_Uptime": OID_SYSUPTIME,
    "System_Contact": OID_SYSCONTACT,
    "System_Location": OID_SYSLOCATION,
    "CPU_Usage": OID_CPUUSAGE,
    "Memory_Usage": OID_MEMORYUSAGE,
    "Disk_Usage": OID_DISKUSAGE,
    "Interface_Count": OID_INTERFACECNT,
}


def find_snmpget():
    import shutil
    path = shutil.which("snmpget")
    if path:
        return path

    for d in [
        r"C:\usr\bin",
        r"C:\usr\local\bin",
        r"C:\Program Files\Net-SNMP\bin",
    ]:
        c = os.path.join(d, "snmpget.exe")
        if os.path.isfile(c):
            return c
    return None


def query_snmp(snmpget, host, oid, version, community):
    """
    Returns:
      {
        "ok": bool,
        "value": str | None,
        "raw": str,
        "error": str | None
      }
    """
    try:
        if version == "2c":
            cmd = [snmpget, "-v", "2c", "-c", community, f"{host}:{SA_SNMP_PORT}", oid]
        else:
            cmd = [snmpget, "-v", "3", "-l", "noAuthNoPriv", "-u", community, f"{host}:{SA_SNMP_PORT}", oid]

        result = subprocess.run(cmd, capture_output=True, text=True, timeout=10)
        stdout = (result.stdout or "").strip()
        stderr = (result.stderr or "").strip()

        if result.returncode == 0 and "=" in stdout:
            value_part = stdout.split("=", 1)[1].strip()
            value = value_part.strip('"').strip("'").strip()
            return {
                "ok": True,
                "value": value,
                "raw": stdout,
                "error": None,
            }

        return {
            "ok": False,
            "value": None,
            "raw": stdout,
            "error": stderr or f"snmpget returncode={result.returncode}",
        }

    except subprocess.TimeoutExpired:
        return {
            "ok": False,
            "value": None,
            "raw": "",
            "error": "timeout",
        }
    except Exception as e:
        return {
            "ok": False,
            "value": None,
            "raw": "",
            "error": str(e),
        }


def collect_sa_data(snmpget):
    """
    Collect System Advisor data and keep diagnostics.
    """
    data = {
        "Timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "SA_Host": SA_HOST,
        "NMS_IP": NMS_IP,
    }

    ok_count = 0
    error_fields = []

    for field_name, oid in OID_MAP.items():
        result = query_snmp(snmpget, SA_HOST, oid, SNMP_VERSION, SNMP_USER)
        data[field_name] = result["value"]

        if result["ok"]:
            ok_count += 1
        else:
            error_fields.append(f"{field_name}:{result['error'] or 'no response'}")

    data["SNMP_OK_Count"] = ok_count
    data["SNMP_Total_OIDs"] = len(OID_MAP)
    data["SNMP_Status"] = "OK" if ok_count > 0 else "ERROR"
    data["SNMP_Error_Summary"] = " | ".join(error_fields[:5]) if error_fields else ""

    return data


def append_to_csv(data):
    try:
        file_exists = DESKTOP_PATH.exists()

        with open(DESKTOP_PATH, "a", newline="", encoding="utf-8") as csvfile:
            fieldnames = list(data.keys())
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)

            if not file_exists:
                writer.writeheader()

            writer.writerow(data)

        return True
    except Exception as e:
        print(f"  *** CSV WRITE ERROR *** {e}")
        return False


def print_collection_summary(count, data, success):
    now = datetime.now().strftime("%H:%M:%S")
    status = "OK" if success else "CSV_ERROR"
    system_name = data.get("System_Name") or "UNKNOWN"
    cpu = data.get("CPU_Usage") or "N/A"
    mem = data.get("Memory_Usage") or "N/A"
    disk = data.get("Disk_Usage") or "N/A"
    snmp_status = data.get("SNMP_Status") or "UNKNOWN"
    ok_count = data.get("SNMP_OK_Count", 0)
    total = data.get("SNMP_Total_OIDs", 0)

    print(
        f"  [{now}] #{count} | {status} | SNMP: {snmp_status} ({ok_count}/{total}) "
        f"| System: {str(system_name)[:25]:<25s} | CPU: {cpu} | Mem: {mem} | Disk: {disk}"
    )

    if data.get("SNMP_Error_Summary"):
        print(f"      Errors: {data['SNMP_Error_Summary']}")


def print_detailed_preview(data):
    print()
    print("  Sample collected data:")
    print("  " + "-" * 64)
    for k, v in data.items():
        print(f"  {k:<22} : {v}")
    print("  " + "-" * 64)
    print()


def main():
    print()
    print("=" * 72)
    print("  SYSTEM ADVISOR DATA COLLECTOR SERVICE")
    print("=" * 72)
    print(f"  System Advisor Host: {SA_HOST}:{SA_SNMP_PORT}")
    print(f"  NMS IP:              {NMS_IP}")
    print(f"  Poll Interval:       {POLL_INTERVAL} seconds")
    print(f"  CSV Output:          {DESKTOP_PATH}")
    print(f"  Started:             {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print("=" * 72)
    print()

    print("  [INIT] Searching for snmpget...")
    snmpget = find_snmpget()
    if not snmpget:
        print("  [INIT] ERROR: snmpget not found.")
        print("  [INIT] Install Net-SNMP and retry.")
        return

    print(f"  [INIT] Using snmpget: {snmpget}")

    print(f"  [INIT] Checking output path: {DESKTOP_PATH}")
    try:
        DESKTOP_PATH.parent.mkdir(parents=True, exist_ok=True)
        print("  [INIT] Output path OK")
    except Exception as e:
        print(f"  [INIT] ERROR: Cannot write to desktop: {e}")
        return

    print(f"  [INIT] Testing SNMP connectivity to {SA_HOST}...")
    test_data = collect_sa_data(snmpget)

    if test_data.get("SNMP_OK_Count", 0) > 0:
        print(f"  [INIT] SNMP OK ({test_data['SNMP_OK_Count']}/{test_data['SNMP_Total_OIDs']} OIDs responded)")
    else:
        print("  [INIT] WARNING: No OIDs responded")
        if test_data.get("SNMP_Error_Summary"):
            print(f"  [INIT] Detail: {test_data['SNMP_Error_Summary']}")

    print_detailed_preview(test_data)

    print("  READY. Collecting System Advisor data... (Ctrl+C to stop)")
    print()

    count = 0
    try:
        while True:
            count += 1
            data = collect_sa_data(snmpget)
            success = append_to_csv(data)
            print_collection_summary(count, data, success)
            time.sleep(POLL_INTERVAL)

    except KeyboardInterrupt:
        print()
        print(f"  Stopped. Collected {count} data point(s).")
        print(f"  Data saved to: {DESKTOP_PATH}")


if __name__ == "__main__":
    main()
