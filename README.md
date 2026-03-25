SELECT radio_id, COUNT(*) AS cantidad
FROM [TRBONET_TEST].[dbo].[Devices]
GROUP BY radio_id
HAVING COUNT(*) > 1
ORDER BY cantidad DESC, radio_id;


SELECT TOP 20 *
FROM [TRBONET_TEST].[dbo].[Devices];
