USE [master];

DECLARE @recordCount INT;

--Select data files
IF OBJECT_ID('tempdb..#FS') IS NOT NULL
    DROP TABLE [#FS];

SELECT DISTINCT [DB].[name]
     , [ALT].[size] / 128.0 AS [CurrentSizeMB]
     , [ALT].[physical_name]
     , [ALT].[database_id]
     , [ALT].[data_space_id]
INTO   [#FS]
FROM   [sys].[master_files] AS [ALT]
JOIN   [sys].[databases] [DB] ON [DB].[database_id] = [ALT].[database_id]
LEFT JOIN  (   SELECT [DB].[name]
                    , [ALT].[size]
               FROM   [sys].[master_files] [ALT]
               JOIN   [sys].[databases] [DB] ON [DB].[database_id] = [ALT].[database_id]) [D] ON [D].[name] = [DB].[name] AND [D].[size] <> [ALT].[size]
WHERE  [ALT].[data_space_id] > 1;


SELECT @recordCount = ISNULL(COUNT(*), 0) 
FROM   [#FS] [M]
JOIN       (   SELECT [name]
                    , [CurrentSizeMB]
                    , [database_id]
                    , [data_space_id]
               FROM   [#FS]) [A] ON [A].[name] = [M].[name] AND [M].[CurrentSizeMB] <> [A].[CurrentSizeMB] AND [A].[data_space_id] = [M].[data_space_id];

IF (@recordCount > 0)
    EXEC [msdb].[dbo].[sp_send_dbmail] 
    @profile_name = 'SQL Alert'
   ,@recipients = 'dba@asicorp.org'
   ,@query = 'SET NOCOUNT ON;
                
                IF OBJECT_ID("tempdb..#FS") IS NOT NULL
                DROP TABLE #FS
                GO 

                SELECT DISTINCT DB.name
                       ,[ALT].[size] / 128.0 AS [CurrentSizeMB]
                       ,[ALT].[physical_name]
                       ,ALT.[database_id]
                       ,ALT.[data_space_id]
                 INTO #FS
                FROM sys.[master_files] AS  ALT
                       JOIN SYS.DATABASES DB ON DB.[database_id] =  ALT.[database_id]
                       LEFT JOIN (SELECT DB.name
                                 ,[ALT].[size] 
                              FROM sys.[master_files] ALT
                              JOIN SYS.DATABASES DB ON DB.[database_id] =  ALT.[database_id]
                                        ) D ON D.Name = DB.Name AND D.Size <> ALT.Size 
                WHERE [ALT].[data_space_id] > 1
                GO 

                SELECT DISTINCT M.Name, M.[CurrentSizeMB], M.[database_id], M.[data_space_id], [M].[physical_name]
                FROM #FS M
                    JOIN (SELECT Name, [CurrentSizeMB], database_id, [data_space_id]
                           FROM #FS) A ON A.Name = M.Name AND M.[CurrentSizeMB] <> A.[CurrentSizeMB] AND A.[database_id] = M.[database_id] AND A.[data_space_id] = M.[data_space_id]
                ORDER BY [M].[physical_name]  
                GO'
    ,@subject = 'ALERT: Different Size Data Files'
    ,@body = 'Adjust data file sizes to be equal.'
    ,@attach_query_result_as_file = 1
    ,@query_attachment_filename = 'UnequalFileSizes_PROD.csv'
    ,@query_result_separator = '	'
    ,@query_result_no_padding = 1;
