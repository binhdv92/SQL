# SQL Snippet
# List SubId string to table
```
DECLARE @SubIdStr VARCHAR(MAX) = '
230904770502
230826731641
'

SET @SubIdStr = REPLACE(REPLACE(@SubIdStr,CHAR(13),' '),CHAR(10),'')
SET @SubIdStr = REPLACE(@SubIdStr,',',' ')
SET @SubIdStr = REPLACE(@SubIdStr,'-',' ')
SET @SubIdStr = REPLACE(@SubIdStr,'_',' ')
SET @SubIdStr = TRIM(@SubIdStr)
IF OBJECT_ID('tempdb..#SubId', 'U') IS NOT NULL BEGIN DROP TABLE #SubId END
SELECT [value] AS SubId INTO #SubId FROM STRING_SPLIT(@SubIdStr,' ') 


```
# Alarm Event History
```
/****** Script for SelectTopNRows command from SSMS  ******/
IF OBJECT_ID('tempdb..#temp', 'U') IS NOT NULL BEGIN DROP TABLE #temp END
SELECT [PdrAlarmEventId]
      ,[EventTypeId]
      ,[SourceLocation]
      ,[EquipmentId]
      ,[EquipmentName]
      ,[Timestamp]
      ,[AlarmEventId]
      ,[AlarmEventName]
      ,[AlarmType]
      ,[AlarmTypeName]
      ,[EventState]
      ,[EventStateName]
      ,[PreviousState]
      ,[PreviousEventStateName]
      ,[LastModifiedTimeUtc]
      ,[LastModifiedUser]
      ,[TimestampUtc]
INTO #temp
  FROM [ODS].[mfg].[PdrAlarmEventDetails]
  WHERE TimeStamp BETWEEN '2023-08-30 09:00:00' AND '2023-08-30 11:10:00'
  AND SourceLocation LIKE 'PGT11%CVS%'
  --AND AlarmEventName LIKE '%3560%' OR AlarmEventName LIKE '%3590%'
  ORDER BY Timestamp

 

  SELECT * FROM #temp
  --WHERE AlarmEventName LIKE '%3560%' OR AlarmEventName LIKE '%3590%'

```
# Merge to database SQL server
```
MERGE INTO MesReporting.[IntRi].[RunTimeYieldPartProducedSummary] AS A
USING(
SELECT [SitePlant], [ReadTimeHour], [WIP_Level], PartProduced FROM #PP_SUM
) AS B ON (A.SitePlant =B.SitePlant AND A.ReadTimeHour = B.ReadTimeHour AND A.WIP_Level = B.WIP_Level)
WHEN MATCHED THEN
UPDATE SET A.PartProduced = B.PartProduced
WHEN NOT MATCHED THEN
INSERT (SitePlant,ReadTimeHour,WIP_Level,PartProduced) VALUES(B.SitePlant, B.ReadTimeHour, B.WIP_Level, B.PartProduced);
```

# Barcode Reader Event
```
SELECT TOP 100 * FROM [ODS].[mfg].[ProcessHistoryPdrBarcodeEvent] BCE
```

# Find Subid which in NCP
```
SELECT  * FROM ModuleAssembly.History.QualityNotifications_NcpHistory
WHERE NcpNumber = '200029574'
```
# find NCP
```
SELECT
    SubId
    , NCP_Dispo
    , NcpNumber
    , DefectDescription
FROM
(
    SELECT
        H.SubId
        , A.Name AS NCP_Dispo
        , ROW_NUMBER() OVER (PARTITION BY H.SubId, H.NcpNumber ORDER BY H.LastModifiedTime DESC) AS RN_DESC
        , H.NcpNumber
        , H.DefectDescription
    FROM 
        ModuleAssembly.History.QualityNotifications_NcpHistory H
            INNER JOIN ModuleAssembly.QualityNotifications.NcpActionTypeEnum A
                ON H.Action = A.NcpActionTypeEnumId
    WHERE
        h.NcpNumber LIKE '%29346%'
) A
WHERE
    RN_DESC = 1
    --AND ncpNumber LIKE '%29346%'
ORDER BY
    A.SubId, A.NcpNumber
```
# UTC convert
```
DECLARE @StartTimeUTC DATETIME 
DECLARE @EndTimeUTC DATETIME 
/* for test */

DECLARE @StartTime DATETIME  = '2022-09-01 00:00:00'
DECLARE @EndTime DATETIME  = '2022-09-02 00:00:00'

BEGIN /* Handle Argument Input */
/* StartTime */
IF @StartTime IS NULL
BEGIN
	SET @StartTime = DATEADD(HOUR, -25, GETDATE())
	SET @StartTime = DATEADD(HOUR, DATEDIFF(HOUR, 0, @StartTime), 0) /* rounded up the time */
END
ELSE
BEGIN
	SET @StartTime = DATEADD(HOUR, DATEDIFF(HOUR, 0, @StartTime), 0) /* rounded up the time */
END
/* EndTime */
IF @EndTime IS NULL
BEGIN
	SET @EndTime = GETDATE()
END
/* Convert UTC Time */
SET @StartTimeUTC = DATEADD(HOUR, -1 * DATEDIFF(HOUR, GETUTCDATE(), GETDATE()),@StartTime )/* Local to GMT */
SET @EndTimeUTC = DATEADD(HOUR,	-1 * DATEDIFF(HOUR, GETUTCDATE(), GETDATE()), @EndTime)/* Local to GMT */
PRINT('time:') PRINT(@StartTime) PRINT(@EndTime) PRINT('timeUTC') PRINT(@StartTimeUTC) PRINT(@EndTimeUTC)
END
```
# Create a temporary table in sql
```IF OBJECT_ID('tempdb..#temp', 'U') IS NOT NULL BEGIN DROP TABLE #temp END```

# Full sameple
```
IF OBJECT_ID('tempdb..#PP', 'U') IS NOT NULL BEGIN DROP TABLE #PP END
SELECT
		[ods].[mfg].[fn_Plant]() AS [SitePlant],
		CAST(DATEADD(HOUR, DATEPART(HOUR, PPE.[TimeStamp]),CAST(CAST(PPE.[TimeStamp] AS DATE) AS DATETIME)) AS DATETIME) AS [ReadTimeHour],
		WIP_Level = ods.sas.WIPLevel(GP.Number),
		GP.[Name] AS ProcessName,
		GP.Number AS ProcessNumber,
		PPE.[Location],
		PPE.ID,
		PPE.[TimeStamp]
INTO	#PP
FROM
		ODS.mfg.ProcessHistoryPdrPartProducedEvent AS PPE
		LEFT OUTER JOIN ODS.mfg.GlobalEquipmentAlias D ON PPE.SourceLocation = D.Alias
		LEFT OUTER JOIN ODS.mfg.GlobalEquipment C ON C.EquipmentId = D.EquipmentId
		LEFT OUTER JOIN ODS.mfg.GlobalProcess GP ON C.ProcessId = GP.ProcessId
WHERE
		[TimeStamp] BETWEEN @StartTime AND @EndTime
		AND ( GP.Number in (
				61100,
				61250,
				61400,
				62342,
				62642,
				63050,
				63675,
				64042,
				64202,
				64417,
				67040)
			OR (GP.Number = 68575 AND (PPE.[Location] like '%INFEED%' OR PPE.[Location] like '%LEG%' OR PPE.[Location] like '%MANUAL')))
		AND NOT(ExitProductStatus = 8)
		AND NOT(SourceLocation = '')
OPTION(max_grant_percent = 10)
```

# HourAgo
```
DECLARE @StartTime DATETIME = GETDATE()
DECLARE @StartTimeAgo DATETIME
DECLARE @HourAgo INT = 5
SET @StartTimeAgo = DATEADD(HOUR, DATEDIFF(HOUR, 0, @StartTime), @HourAgo) /* rounded up the time */
SELECT @StartTimeAgo, @StartTime
```
# WW Ago
```WWAgo = ods.sas.ISOWTDShort(GETDATE()) - ods.sas.ISOWTDShort([ReadTime])```

# Sample code for Date Time Hierarchy
```
	ISOYear = ods.sas.IsoYear(MS.ReadTime),
    CONCAT(CONVERT(VARCHAR(4), DATEPART(YEAR, MS.ReadTime)), '_', RIGHT( '0' + CONVERT(VARCHAR(2), DATEPART(QUARTER, MS.ReadTime)), 2)) AS [QTD],
    CONCAT(CONVERT(VARCHAR(4), DATEPART(YEAR, MS.ReadTime)), '_', RIGHT( '0' + CONVERT(VARCHAR(2), DATEPART(MONTH, MS.ReadTime)), 2)) AS [MTD],
    CONCAT(CONVERT(VARCHAR(4), DATEPART(YEAR, MS.ReadTime)), '_', RIGHT( '0' + CONVERT(VARCHAR(2), DATEPART(WEEK, MS.ReadTime)), 2)) AS [WTD],
	ISOQTD = ods.sas.ISOQTD(MS.ReadTime),
	ISOWTD = ods.sas.ISOWTD(MS.ReadTime),
    [ods].[sas].[getcrewfull](ods.mfg.fn_Plant(), MS.ReadTime) AS [Shift],
    [ods].[sas].[GetCrew](ods.mfg.fn_Plant(), MS.ReadTime) AS [ShiftName],
    CONVERT(DateTime, DATEADD(dd, DATEDIFF(dd, 0, [ReadTime]), 0)) AS [DateTime],*/
    CAST(DATEADD(HOUR, DATEPART(HOUR, MS.ReadTime), CAST(CAST(MS.ReadTime AS DATE) AS DATETIME)) AS DATETIME) AS [ReadTimeHour],
    CONVERT(DATETIME, MS.ReadTime) AS ReadTime,
```

# Current Crew
```
DECLARE @today DATETIME= DATEADD(HOUR, DATEDIFF(HOUR, 0, GETDATE()), 0)
SELECT
GETDATE() AS LastRefreshedTimestamp
, CONCAT('Current Crew: ',ods.sas.GetCrew(ods.mfg.fn_Plant(),DATEADD(HOUR,0,GETDATE()))) AS ShiftAgo0
, CONCAT('Last Crew: ',ods.sas.GetCrew(ods.mfg.fn_Plant(),DATEADD(HOUR,-12,GETDATE()))) AS ShiftAgo1
, CONCAT('2 Crews Ago: ',ods.sas.GetCrew(ods.mfg.fn_Plant(),DATEADD(HOUR,-24,GETDATE()))) AS ShiftAgo2

```

# Last Status of EquipmentName:
```
/****** Script for SelectTopNRows command from SSMS  ******/
IF OBJECT_ID('tempdb..#table1', 'U') IS NOT NULL BEGIN DROP TABLE #table1 END
SELECT TOP (1000) [PdrEquipmentStateId]
      ,[EventTypeId]
      ,[SourceLocation]
      ,[TimestampUtc]
      ,[EquipmentState]
      ,[PreviousState]
      ,[PreviousStateLength]
      ,[ChangedBy]
      ,[ChangeReason]
      ,[ChangeComment]
      ,[LastModifiedTimeUtc]
      ,[LastModifiedUser]
      ,[TimeStamp]
INTO #table1
  FROM [ModuleAssembly].[ProcessHistory].[PdrEquipmentState]
  WHERE TimeStamp BETWEEN GETDATE()-1 AND GETDATE()

		AND SourceLocation LIKE 'PGT1__-VTD_COATER'


SELECT *
FROM (
SELECT * 
, 
RANK() OVER (PARTITION BY [SourceLocation] ORDER BY TimeStamp DESC) AS Flag_Rank01
FROM #table1
) AS A
WHERE Flag_Rank01 = 1
ORDER BY SourceLocation, TimeStamp DESC


```


# Looping Calendar
```

/* Switch on statistics time */
DECLARE @Statistic DATETIME = GETDATE()

/**************************************************************************************/
/*** Introduction ***/
/**************************************************************************************/
/***
	Title: 
		uspRunTimePartProducedSummary
	Version:
		v1.0.0
	Author:
		van.binh.duong@firstsolar.com
	Created Date: 2022-12-11
	Modified Date: ---
	Noted:
		- 
	Traching History:
		2022-12-11: Created.

***/

/*** for testing ***/

DECLARE @StartTime DATETIME = '2022-12-01 00:00:00'
DECLARE @EndTime	DATETIME = '2022-12-10 00:00:00'
DECLARE @OffsetIncrease INT = 24


PRINT('/********************************************************************/')
PRINT('Configuration')
PRINT('/********************************************************************/')
SET DATEFIRST 1
SET NOCOUNT OFF
DECLARE @StartTimeUtc DATETIME
DECLARE @EndTimeUtc DATETIME
PRINT(CONCAT('@OffsetIncrease: ',@OffsetIncrease))

PRINT('/********************************************************************/')
PRINT('Setting up Arguments')
PRINT('/********************************************************************/')
/*** StartTime ***/
IF @StartTime IS NULL
	BEGIN
		SET @StartTime = DATEADD(HOUR, -3, GETDATE())
		SET @StartTime = DATEADD(HOUR, DATEDIFF(HOUR, 0, @StartTime), 0) /* rounded up the time */
	END
ELSE
	BEGIN
		SET @StartTime = DATEADD(HOUR, DATEDIFF(HOUR, 0, @StartTime), 0) /* rounded up the time */
	END
/*** EndTime ***/
IF @EndTime IS NULL
	BEGIN
		SET @EndTime = GETDATE()
		SET @EndTime = DATEADD(HOUR, DATEDIFF(HOUR, 0, @EndTime)+1, 0) /* rounded up the time */
	END
ELSE
	BEGIN
		SET @EndTime = DATEADD(HOUR, DATEDIFF(HOUR, 0, @EndTime), 0) /* rounded up the time */
	END

/* Convert UTC Time */
SET @StartTimeUTC	= DATEADD(HOUR, -1 * DATEDIFF(HOUR, GETUTCDATE(), GETDATE()), @StartTime)	/*** Local to GMT ***/
SET @EndTimeUTC		= DATEADD(HOUR,	-1 * DATEDIFF(HOUR, GETUTCDATE(), GETDATE()), @EndTime)		/*** Local to GMT ***/
PRINT('time:') PRINT(@StartTime) PRINT(@EndTime) PRINT('timeUTC') PRINT(@StartTimeUTC) PRINT(@EndTimeUTC)


PRINT('/*********/')
PRINT('/*** #Calendar ***/')
PRINT('/*********/')
IF OBJECT_ID('tempdb..#Calendar', 'U') IS NOT NULL BEGIN DROP TABLE #Calendar END
CREATE TABLE #Calendar(
	[ID] INT IDENTITY(1,1) NOT NULL,
	[StartTime] Datetime,
	[EndTime] DateTime
)

DECLARE @iStartTime DATETIME
DECLARE @iEndTime DATETIME
SET @iStartTime = @StartTime
WHILE @iStartTime <= @EndTime
BEGIN
	SET @iEndTime = DATEADD(HOUR, @OffsetIncrease, @iStartTime)
	INSERT INTO #Calendar([StartTime], [EndTime]) VALUES(@iStartTime,@iEndTime)
	SET @iStartTime = DATEADD(HOUR, @OffsetIncrease, @iStartTime)
END

PRINT('/********************************************************************/')
PRINT('Preparing LOOPING')
PRINT('/********************************************************************/')
DECLARE @iStartTime2 DATETIME
DECLARE @iEndTime2 DATETIME
DECLARE @iRow INT, @nROw INT
SELECT @nRow = COUNT(0) FROM #Calendar
PRINT(CONCAT('There is total (@nRow) ', @nRow, ' loops'))
PRINT(CONCAT('@OffsetIncrease: ', @OffsetIncrease))
PRINT(CONCAT('@StartTime: ', @StartTime, '; @EndTime:',@EndTime))
SET @iRow = @nRow
WHILE @iRow >= 0
BEGIN
	SELECT @iStartTime2 = StartTime, @iEndTime2 = EndTime FROM #Calendar WHERE ID = @iRow
	PRINT(CONCAT(@iRow, '/', @nRow, ' ', @iStartTime2,' ',@iEndTime2))

	/*** Your Main Scrip Here ***/
	
	/*** End Your Main Scrip Here ***/
	SET @iRow = @iRow - 1
END


PRINT('/********************************************************************/')
PRINT('The End!')
PRINT('/********************************************************************/')
/* Switch off statistics time */
PRINT(CONCAT('Total Execution Time: ',CAST(GETDATE() - @Statistic AS TIME)))


```
