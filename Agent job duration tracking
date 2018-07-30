SELECT 
    J.name as 'JobName',
    S.step_id as 'Step',
    S.step_name as 'StepName',
    msdb.dbo.agent_datetime(run_date, run_time) as 'RunDateTime',
    CAST((run_duration/10000) AS varchar) + ':' + CAST((run_duration/100%100) AS varchar) + ':' + CAST((run_duration%100) AS varchar) AS 'RunDuration',
    ((run_duration/10000*3600 + (run_duration/100)%100*60 + run_duration%100 + 31 ) / 60) AS 'RunDurationMinutes'
FROM msdb.dbo.sysjobs J 
    JOIN msdb.dbo.sysjobsteps S ON J.job_id = S.job_id
    JOIN msdb.dbo.sysjobhistory H ON S.job_id = H.job_id AND S.step_id = H.step_id 
WHERE J.enabled = 1   --Only Enabled Jobs
    AND msdb.dbo.agent_datetime(run_date, run_time) BETWEEN GETDATE() - 1 AND GETDATE() --Select timeframe for job runs
    AND J.name = <Enter Job Name>
    AND S.step_name = <Enter Step Name>
ORDER BY JobName, RunDateTime DESC
