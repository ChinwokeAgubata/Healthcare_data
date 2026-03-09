Project Overview
​This project involved building an end-to-end data pipeline to analyze hospital operations and financial metrics. I transformed raw, complex healthcare data into a high-level executive dashboard tracking 1.40 billion in total billing across 39,876 hospitals.
1. Data Cleaning  (SQL Server / SSMS)
Before visualization, I managed the raw data in SSMS to ensure it was accurate and optimized. This was the most critical phase of the project.

Name Capitalization: I capitalized the first letter of the first and last name and also removed trailing inconsistent spaces between the name.
SELECT 
    Name AS Original_Name,
    -- Capitalize First Name
    UPPER(LEFT(Name, 1)) + 
    LOWER(SUBSTRING(Name, 2, CHARINDEX(' ', Name + ' ') - 2)) + ' ' +
    -- Capitalize Last Name

    UPPER(SUBSTRING(Name, CHARINDEX(' ', Name + ' ') + 1, 1)) + 
    LOWER(SUBSTRING(Name, CHARINDEX(' ', Name + ' ') + 2, LEN(Name))) AS Cleaned_Name
FROM dbo.healthcare_dataset; 

UPDATE dbo.healthcare_dataset
SET Name = UPPER(LEFT(Name, 1)) + 
           LOWER(SUBSTRING(Name, 2, CHARINDEX(' ', Name + ' ') - 2)) + ' ' +
           UPPER(SUBSTRING(Name, CHARINDEX(' ', Name + ' ') + 1, 1)) + 
           LOWER(SUBSTRING(Name, CHARINDEX(' ', Name + ' ') + 2, LEN(Name)));


Trimming Hidden Spaces: For each column, hidden spaces were trimmed using the following queries.
UPDATE dbo.healthcare_dataset
SET Gender = TRIM(Gender),
    Blood_Type = TRIM(Blood_Type),
    Medical_Condition = TRIM(Medical_Condition),
    Insurance_Provider = TRIM(Insurance_Provider);

Standardizing medical condition and blood type: the first letter of every medical condition was capitalized using the folloqing sql query
UPDATE dbo.healthcare_dataset
SET Medical_Condition = UPPER(LEFT(Medical_Condition, 1)) + LOWER(SUBSTRING(Medical_Condition, 2, LEN(Medical_Condition)));

Handling Impossible dates: To prevent inconsistensies in futher calculations ie determining hospital length of stay, we cheked and ensured that the discharged date was farther or the same date as the admission date and not before.
SELECT * FROM dbo.healthcare_dataset 
WHERE Discharge_Date < Date_of_Admission; 

Rounding billing Amount: The billing amount was rounded to two decimal places, to ensure data consistencies .
UPDATE dbo.healthcare_dataset
SET Billing_Amount = ROUND(Billing_Amount, 2);


De-duplication with CTEs: I used Common Table Expressions (CTEs) and the ROW_NUMBER() function to identify and remove duplicate patient records. This ensured my final patient count of 54,966 was precise.
-- Finding Duplicates
SELECT Name, Age, Gender, Blood_Type, Medical_Condition, Date_of_Admission,
Doctor,
Hospital,
Insurance_Provider,
Billing_Amount,
Room_Number,
Admission_Type,
Discharge_Date,
Medication,
Test_Results,
 COUNT(*) as Duplicate_Count
FROM dbo.healthcare_dataset
GROUP BY Name,Age, Gender, Blood_Type, Medical_Condition, Date_of_Admission, Doctor, Hospital, Insurance_Provider, Billing_Amount, Room_Number, Admission_Type, Discharge_Date, Medication, Test_Results
HAVING COUNT(*) > 1;

-- Deleting Duplicates using CTE and a window function called Row_Number 
WITH CTE AS (
    SELECT *, 
    ROW_NUMBER() OVER (
        PARTITION BY Name, Age, Gender, Blood_Type, Medical_Condition, Date_of_Admission, Doctor, Hospital, Insurance_Provider, Billing_Amount, Room_Number, Admission_Type, Discharge_Date, Medication, Test_Results
        ORDER BY Name
    ) as RowNumber
    FROM dbo.healthcare_dataset
)


2.Tables: Tables were created from this heathcare data set to make visualization easily done in Power Bi.

Executive summary
I used the sql functions ‘COUNT’, ‘ROUND’, ‘SUM’, ‘AVG’, ‘DATEDIFF’ to show the hospital performance at a glance. ie Unique_Patients, Total_Revenue and Avg_stay_duration.
SELECT 
    Hospital, 
    COUNT(DISTINCT Patient_Row_ID) AS Unique_Patients,
    ROUND(SUM(Billing_Amount), 2) AS Total_Revenue,
    AVG(DATEDIFF(day, Date_of_Admission, Discharge_Date)) AS Avg_Stay_Duration
FROM dbo.healthcare_dataset
GROUP BY Hospital
ORDER BY Total_Revenue DESC;


Patient Journal
The column Patient_Row_Id, with the sql function ‘LAG’ was used to track the history of a single person across different visits ie for each person was there a previous condition that prompted your visit to the hospital?

SELECT 
    Patient_Row_ID, 
    Name, 
    Medical_Condition, 
    Date_of_Admission, 
    Medication,
    LAG(Medical_Condition) OVER (PARTITION BY Name, Age ORDER BY Date_of_Admission) AS Previous_Condition
FROM dbo.healthcare_dataset;


Admission type trend
This table displays the admission type along side the Hospital, Patient_Row_Id and Insurance Provder.
SELECT distinct(Hospital), Insurance_Provider, Admission_Type, COUNT(*) as Count, Patient_Row_ID
FROM dbo.healthcare_dataset
GROUP BY Hospital, Insurance_Provider, Admission_Type,Patient_Row_ID
ORDER BY Admission_Type,Count DESC;


