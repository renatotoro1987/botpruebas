;WITH duplicados AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY radio_id
               ORDER BY id DESC
           ) AS rn
    FROM [TRBONET_TEST].[dbo].[Devices]
)
SELECT *
FROM duplicados
WHERE rn > 1
ORDER BY radio_id, id DESC;



;WITH duplicados AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY radio_id
               ORDER BY id DESC
           ) AS rn
    FROM [TRBONET_TEST].[dbo].[Devices]
)
SELECT *
INTO [TRBONET_TEST].[dbo].[Devices_Duplicados_Backup_20260325]
FROM duplicados
WHERE rn > 1;





;WITH duplicados AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY radio_id
               ORDER BY id DESC
           ) AS rn
    FROM [TRBONET_TEST].[dbo].[Devices]
)
SELECT COUNT(*) AS registros_a_borrar
FROM duplicados
WHERE rn > 1;




BEGIN TRANSACTION;

;WITH duplicados AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY radio_id
               ORDER BY id DESC
           ) AS rn
    FROM [TRBONET_TEST].[dbo].[Devices]
)
DELETE FROM duplicados
WHERE rn > 1;

SELECT @@ROWCOUNT AS filas_borradas;

COMMIT TRANSACTION;




SELECT radio_id, COUNT(*) AS cantidad
FROM [TRBONET_TEST].[dbo].[Devices]
GROUP BY radio_id
HAVING COUNT(*) > 1
ORDER BY cantidad DESC, radio_id;
