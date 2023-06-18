## Azure Stream Analytics

### Introduction

This is an azure analytics demonstration for an atm system. The system is composed by 4 main components:
* Atm input data that come from a event hub
* A stream analytics job that process the data
* Blob storage that stores the results
* Reference data that contains the atm locations, customers and locations

## Setup

Below you can see the architecture of the system:

![Screenshot_121](https://github.com/maarioos1308/AzureStreamAnalytics/assets/93925338/0164e77e-9b0d-496d-a872-7c2e8f4e6b1b)



## Query

### Query 1

Show the total “Amount” of “Type = 0” transactions at “ATM Code = 21” of the last 10 minutes. Repeat as new events
keep flowing in (use a sliding window).

```sql
SELECT
    System.Timestamp AS StartTime,
    SUM(Amount) AS TotalAmount
INTO [output1]
FROM [DataInput] 
WHERE Type = 0 AND ATMCode = 21
GROUP BY SlidingWindow(minute, 10)
```
The query output looks like this:
<br>"StartTime":"2023-06-18T12:56:22.7170000Z","TotalAmount":34.0</br>
"StartTime":"2023-06-18T13:06:42.6820000Z","TotalAmount":49.0

### Query 2
Show the total “Amount” of “Type = 1” transactions at “ATM Code = 21” of the last hour. Repeat once every hour
(use a tumbling window).

```sql
SELECT
    System.Timestamp AS StartTime,
    SUM(DataInput.Amount) AS TotalAmount
INTO [output2]
FROM [DataInput]
WHERE Type = 1 AND ATMCode = 21
GROUP BY TumblingWindow(hour, 1) 
```
The query output looks like this:
<br>{"StartTime":"2023-06-18T16:00:00.0000000Z","TotalAmount":35.0}</br>
{"StartTime":"2023-06-18T17:00:00.0000000Z","TotalAmount":57.0}

### Query 3
Show the total “Amount” of “Type = 1” transactions at “ATM Code = 21” of the last hour. Repeat once every 30 minutes
(use a hopping window)
    
```sql
SELECT
    System.Timestamp AS StartTime,
    SUM(Amount) AS TotalAmount
INTO output3
FROM DataInput
WHERE Type = 1 AND ATMCode = 21
GROUP BY HoppingWindow(minute, 60, 30)
```

The query output looks like this:
<br>{"StartTime":"2023-06-18T16:00:00.0000000Z","TotalAmount":35.0}</br>
{"StartTime":"2023-06-18T16:30:00.0000000Z","TotalAmount":57.0}

### Query 4
Show the total “Amount” of “Type = 1” transactions per “ATM Code” of the last one hour (use a sliding window)
    
```sql
SELECT
    System.Timestamp AS StartTime,
    DataInput.ATMCode AS ATM_Code,
    SUM(DataInput.Amount) AS TotalAmount   
INTO [output4]
FROM DataInput
WHERE DataInput.Type = 1
GROUP BY DataInput.ATMCode, SlidingWindow(hour, 1)
```

The query output looks like this:
<br>{"StartTime":"2023-06-18T16:00:00.0000000Z","ATM_Code":21,"TotalAmount":35.0}</br>
{"StartTime":"2023-06-18T16:00:00.0000000Z","ATM_Code":22,"TotalAmount":57.0}

### Query 5
Show the total “Amount” of “Type = 1” transactions per “Area Code” of the last hour. Repeat once every hour (use a
tumbling window)
    
```sql
SELECT
    System.Timestamp AS StartTime,
    atm.area_code AS AreaCode,
    SUM(DataInput.Amount) AS TotalAmount
INTO output5
FROM DataInput
JOIN atm
    ON DataInput.AtmCode = atm.atm_code
WHERE DataInput.Type = 1
GROUP BY atm.area_code, TumblingWindow(hour, 1)
```

The query output looks like this:
<br>{"StartTime":"2023-06-18T16:00:00.0000000Z","AreaCode":1,"TotalAmount":35.0}</br>
{"StartTime":"2023-06-18T17:00:00.0000000Z","AreaCode":2,"TotalAmount":57.0}

### Query 6
Show the total “Amount” per ATM’s “City” and Customer’s “Gender” of the last hour. Repeat once every hour (use a
tumbling window)
    
```sql
SELECT
    System.Timestamp AS StartTime,
    area.area_city AS ATM_City,
    customer.gender AS Customer_Gender,
    SUM(DataInput.Amount) AS TotalAmount
INTO [output6]
FROM [DataInput] 
JOIN atm
    ON DataInput.ATMCode = atm.atm_code
JOIN customer
    ON DataInput.CardNumber = customer.card_number
JOIN Area
    ON atm.area_code = area.area_code
GROUP BY area.area_city, customer.gender, TumblingWindow(hour, 1)
```

The query output looks like this:
{"StartTime":"2023-06-18T13:00:00.0000000Z","ATM_City":"Schaumburg","Customer_Gender":"Female","TotalAmount":1959.0}
{"StartTime":"2023-06-18T13:00:00.0000000Z","ATM_City":"Baltimore","Customer_Gender":"Male","TotalAmount":343.0}
{"StartTime":"2023-06-18T13:00:00.0000000Z","ATM_City":"Omaha","Customer_Gender":"Female","TotalAmount":182.0}
{"StartTime":"2023-06-18T13:00:00.0000000Z","ATM_City":"Tacoma","Customer_Gender":"Male","TotalAmount":16.0}
{"StartTime":"2023-06-18T13:00:00.0000000Z","ATM_City":"Memphis","Customer_Gender":"Male","TotalAmount":1187.0}
{"StartTime":"2023-06-18T13:00:00.0000000Z","ATM_City":"Tacoma","Customer_Gender":"Female","TotalAmount":719.0}

### Explanation
In order query 7 and query 8 to work we used <code>WITH</code> clause to create tables that will be used in the queries.
You can see below the tables that we created:

```sql
WITH 
AlertData AS (
    SELECT
        input.ATMCode,
        atm.area_code AS ATM_AreaCode,
        customer.area_code AS Customer_AreaCode  
    FROM [input]
    JOIN atm
        ON input.ATMCode = atm.atm_code
    JOIN customer
        ON input.CardNumber = customer.card_number
    WHERE atm.area_code <> customer.area_code
),
TransactionCounts AS (
  SELECT
    CardNumber,
    COUNT(*) AS TransactionCount
  FROM
    input
  WHERE
    Type = 1
  GROUP BY
    SlidingWindow(hour, 1), CardNumber
)
```

### Query 7
Alert (SELECT “1”) if a Customer has performed two transactions of “Type = 1” in a window of an hour (use a sliding
window)
    
```sql
SELECT 
    System.Timestamp AS StartTime,
    'Alert' AS Message
INTO output7
FROM TransactionCounts
WHERE TransactionCount >= 2
```

The query output looks like this:
<br>{"StartTime":"2023-06-18T16:00:00.0000000Z","Message":"Alert"}</br>
{"StartTime":"2023-06-18T16:00:00.0000000Z","Message":"Alert"}

### Query 8
Alert (SELECT “1”) if the “Area Code” of the ATM of the transaction is not the same as the “Area Code” of the “Card
Number” (Customer’s Area Code) - (use a sliding window)
        
```sql
SELECT
System.Timestamp AS StartTime,
MAX('1') AS Alert
INTO [output8]
FROM AlertData
GROUP BY SlidingWindow(hour, 1)
```

The query output looks like this:
<br>{"StartTime":"2023-06-18T16:00:00.0000000Z","Alert":"1"}</br>
{"StartTime":"2023-06-18T16:00:00.0000000Z","Alert":"1"}

## Authors
* Marios Ajdini (8200003) 
* Giorgos Zarkadas (8200043)
