# kpi_query_manual.py
import datetime as dt
import pyodbc
import requests

SQL_CONN = (
    "DRIVER={ODBC Driver 17 for SQL Server};"
    "SERVER=10.109.85.129,1433;"
    "DATABASE=GW;"
    "UID=RTORO;"
    "PWD=Motorola1;"
    "Encrypt=no;"
    "TrustServerCertificate=yes;"
    "Connection Timeout=20;"
)

SEND_TO_VPS = True
API_LOGIN_URL = "https://monitor.serviciosbull.cl/api/auth/login"
API_PUSH_URL = "https://monitor.serviciosbull.cl/api/kpi/daily/busy"
API_USER = "admin"
API_PASS = "Admin2026!"

# FECHA QUE QUIERES CARGAR (manual)
TARGET_DATE = dt.date(2026, 2, 26)

fecha_ini = f"{TARGET_DATE.isoformat()}T00:00:00"
fecha_fin = f"{TARGET_DATE.isoformat()}T23:59:59"

QUERY = f"""
SET NOCOUNT ON;

DECLARE @FechaIni datetime = '{fecha_ini}';
DECLARE @FechaFin datetime = '{fecha_fin}';
DECLARE @FechaFinAbierta datetime = DATEADD(SECOND, 1, @FechaFin);

;WITH
SitesTable (SiteId, SiteAlias) AS (
    SELECT
        s.ID AS SiteId,
        s.Alias AS SiteAlias
    FROM dbo.RES_Sites_Full s
    WHERE s.SiteUniq IN (
        SELECT ID
        FROM dbo.RPT_ReportParameters WITH (NOLOCK)
        WHERE [Source] = 'GW3MTRBO_RTORO' AND [Type] = 6
    )
),
Horas (sdt, edt) AS (
    SELECT
        CAST(@FechaIni AS datetime) AS sdt,
        DATEADD(HOUR, 1, CAST(@FechaIni AS datetime)) AS edt
    UNION ALL
    SELECT
        DATEADD(HOUR, 1, sdt),
        DATEADD(HOUR, 2, sdt)
    FROM Horas
    WHERE DATEADD(HOUR, 1, sdt) < @FechaFinAbierta
),
CallsHora AS (
    SELECT
        h.sdt,
        c.SiteId,
        COUNT_BIG(*) AS CantLlamadas
    FROM dbo.ARC_Calls_ReportView c
    JOIN Horas h
      ON c.StartDT >= h.sdt AND c.StartDT < h.edt
    WHERE c.StartDT >= @FechaIni
      AND c.StartDT <  @FechaFinAbierta
    GROUP BY
        h.sdt,
        c.SiteId
),
BusiesHora AS (
    SELECT
        h.sdt,
        b.SiteId,
        COUNT_BIG(*) AS CantBusys
    FROM dbo.ARC_Busies_ReportView b
    JOIN Horas h
      ON b.StartDT >= h.sdt AND b.StartDT < h.edt
    WHERE b.StartDT >= @FechaIni
      AND b.StartDT <  @FechaFinAbierta
    GROUP BY
        h.sdt,
        b.SiteId
)
SELECT
    CAST(h.sdt AS date) AS Fecha,
    CONVERT(char(5), h.sdt, 108) AS Hora,
    s.SiteId AS SiteID,
    s.SiteAlias,
    CAST(ISNULL(c.CantLlamadas, 0) AS bigint) AS CantLlamadas,
    CAST(ISNULL(b.CantBusys, 0) AS bigint) AS CantBusys
FROM SitesTable s
CROSS JOIN Horas h
LEFT JOIN CallsHora c
  ON c.sdt = h.sdt AND c.SiteId = s.SiteId
LEFT JOIN BusiesHora b
  ON b.sdt = h.sdt AND b.SiteId = s.SiteId
ORDER BY
    s.SiteAlias, Fecha, Hora
OPTION (MAXRECURSION 0);
"""

def rows_to_dicts(cur):
    cols = [c[0] for c in cur.description]
    out = []
    for row in cur.fetchall():
        d = {}
        for i, v in enumerate(row):
            if isinstance(v, (dt.datetime, dt.date, dt.time)):
                d[cols[i]] = v.isoformat()
            else:
                d[cols[i]] = v
        out.append(d)
    return out

def get_api_token():
    r = requests.post(API_LOGIN_URL, json={"username": API_USER, "password": API_PASS}, timeout=20)
    r.raise_for_status()
    return r.json()["access_token"]

def main():
    print("Conectando SQL...")
    with pyodbc.connect(SQL_CONN, timeout=20) as cn:
        cur = cn.cursor()
        print("Ejecutando consulta...")
        cur.execute(QUERY)
        rows = rows_to_dicts(cur)

    print(f"Filas obtenidas: {len(rows)}")
    if rows:
        print("Muestra fila 1:", rows[0])

    if not SEND_TO_VPS:
        print("SEND_TO_VPS=False, no se envia al VPS.")
        return

    payload = {
        "kpi_date": TARGET_DATE.isoformat(),  # <- ahora coincide con la fecha consultada
        "source": "GW3MTRBO_RTORO",
        "summary": rows,
        "chart": rows[:500],
    }

    token = get_api_token()
    headers = {"Authorization": f"Bearer {token}"}

    print("Enviando al VPS...")
    r = requests.post(API_PUSH_URL, json=payload, headers=headers, timeout=60)
    print("HTTP:", r.status_code)
    print("BODY:", r.text)

if __name__ == "__main__":
    main()
