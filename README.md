SELECT radio_id, COUNT(*) AS cantidad
FROM dbo.Devices
GROUP BY radio_id
HAVING COUNT(*) > 1
ORDER BY cantidad DESC;
