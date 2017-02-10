# Knowledge Base
My knowledge base of Technical Interview Questions
 
### ASP.NET, MVC, Testing
1. How to Handle a MVC multi user site scenario where one page retrieves its data (View page) from say a staging table (Flat file table that say is a join of a master table and foreign key tables) for fast retrieval and another page may be used for updating the underling data (Edit page) the staging table is based on, so how do you keep the staging table synchronised in the most efficient way?
 * On the View page have a timer event that refreshes the data via a partial view using an ajax call to the server asynchronously. In that I am guranteed to have up-to-date data on my screen
 * On the Edit page on the Server when saving: update both the underlying table, the staging table and have a flag to indicate the table has been edited say Application["TableIsDirty"] = true; so that all users can detect it when the View page is refreshed; it can check if the application variable is true whether to use data from cache or to refresh then unset the flag i.e. Application["TableIsDirty"] = false, important to surround your code with a lock system so that 2 users cannot enter the critical area at the same time, whenever the View page is reached or reloaded by a subsequent user the data is always retrived from the cache. This way any number of users can be editing differnt rows or viewing data.
 * Alternatively you could go one step further depending how much data needs to retrieved and whether it is more effecient to load a single row only, then in your Edit page assuming the table has an ID key field you could collect an array of ID's of those rows beeing updated as in Application["P" + ID.ToString()] = true, then your code block in your View page can loop thru this array to create a filter on all ID's so only retrieve the records that have been changed; then I suggest rather than setting to false remove all the ID's from the application varible collection

2. Handle a database deadlock in big set of running tests:
Use a Timeout locking method so it at least releases one of calls then it can log a message the method it occurred in (e.g. ); which as soon as there is a deadlock stop and throw an error) there you trap the error and can pin down the particular database call causing the problem, perhaps you could reorder that particular test to avoid a future lock to see if the problem disappears, then you can safely assume it is a just purely a consequence of you testing as supposed to a production error
3. Global handling of unhandled errors server side code:
Error raised in the Global.asax file which you can intercept and further
Use Server.GetLastError to get the HttpExcpetion, there you add logging and/or set a session flag so you can read in a 
a customer error page or reload of the same page. You can set the Session then call Server.ClearError to clear last error and stop .net from clearing the new session
4. Javascript handling of errors say in an ajax call:
There is no specific event client side so you have to handle your exceptions in every function however returning code from the server in an ajax call you can make use of callback onerror event handler to notify the user.

### SQL
1. Joining on non mandatory master/child 1 to many relationship tables to display only rows where there is no child relationship

Option 1 - BAD
```
SELECT m.*
FROM MasterTable m
WHERE ID NOT IN (SELECT c.MasterID FROM ChildTable c)
```

Why is the above SQL wrong?
Well, the query will actually give you the correct result however; it is probably the least effecient answer.
For every row in the master table it has to go through the same process of traversing through the entire Child table to see if there is a match, which will be a long process especially if the ChildTable is huge.

Option 2 - GOOD
 ```
SELECT m.*
FROM MasterTable m
LEFT OUTER JOIN ChildTable c
  ON m.ID = c.MasterID
WHERE c.MasterID IS NULL
```
Here by using a join between the two tables the process is aligant and most effecient - we can assume (best practice is the ID column is the primary key and thus the clustered index) that both tables are pre-ordered, so then the first action is to just sort the child table by Master ID and only have distinct values, so then all the process requires is one traverse of the master and child tables together as supposed to many look-ups into the child table that the first solution would do.

2. One of my now new favourite SQL questions is how do you write a query to display a list of employees and their immediate manager. A typical example is the Employee/Manager scenario where the Employee table has a ManagerID column and a manager is also an employee and you want to specifically show the Manager's name in your query not just the ID and group employees by organisation structure.

There are actually 2 good answers to this question the first is to use an old technique but a good one called recursion and this can be achieved in T-SQL using a common table expression (CTE), in order to order the data in a tree like organisation chart structure you can use another techiniue in your recursion process to append the child employeeid to the its manager's NodeID and use a separator character between to distinguish hierachy and thus you get a path from the bottom to the top of the tree. The second answer is thanks to a new datatype in SQL2008 hierarchyid which I will explain only briefly solves the problem of the first solution being somewhat complicated. The full explanation of how the hierachyid works is left as an exercise for the reader to research.

### Solution 1 - recursion thru CTE

Data

| EmployeeID | LoginID | ManagerID | Title |
| --- | --- | --- | --- |
| 1 | Donald Trump | NULL | President |
| 2 | Mike Pencetille | 1 | Vice President |
| 3 | Jared Kushner | 1 | Senior Advisor |
| 4 | Ivanka Kushner | 3 | Wife of Senior Advisor and daughter of Presiodent |
| 5 | VP Helper | 2 | Helper to Vice President |

```
WITH Employee AS (
	SELECT 1 AS [EmployeeID]
      ,CAST(N'Donald Trump' AS NVARCHAR(256)) AS LoginID
      ,CAST(NULL AS INT) AS [ManagerID]
      ,CAST(N'President' AS NVARCHAR(50)) AS [Title]
	
	UNION
	
	SELECT 2 AS EmployeeID
      ,N'Mike Pencetille' AS LoginID
      ,1 AS [ManagerID]
      ,N'Vice President' AS [Title]
	
	UNION
	
	SELECT 3 AS EmployeeID
      ,N'Jared Kushner' AS LoginID
      ,1 AS [ManagerID]
      ,N'Senior Advisor' AS [Title]
	
	UNION
	
	SELECT 4 AS EmployeeID
      ,N'Ivanka Kushner' AS LoginID
      ,3 AS [ManagerID]
      ,N'Wife of Senior Advisor and daughter of Presiodent' AS [Title]
	
	UNION
	
	SELECT 5 AS EmployeeID
      ,N'VP Helper' AS LoginID
      ,2 AS [ManagerID]
      ,N'Helper to Vice President' AS [Title]
), EmployeeList AS (
	SELECT Boss.[EmployeeID]
      ,Boss.[LoginID]
      ,Boss.[ManagerID], CAST(NULL AS NVARCHAR(256)) AS ManagerLoginID
      ,Boss.[Title]
	    ,1 AS EmployeeLevel
	    ,CAST(Boss.[EmployeeID] AS NVARCHAR(256)) AS NodeID
	FROM Employee Boss
	WHERE Boss.ManagerID IS NULL

	UNION ALL

	SELECT Emp.[EmployeeID]
      ,Emp.[LoginID]
      ,Emp.[ManagerID], EL.LoginID
      ,Emp.[Title]
	    ,EL.EmployeeLevel + 1
	    ,CAST(EL.NodeID + '\' + CAST(Emp.EmployeeID AS NVARCHAR) AS NVARCHAR(256))
	FROM Employee Emp
	INNER JOIN EmployeeList EL ON Emp.ManagerID = EL.EmployeeID
	WHERE Emp.ManagerID IS NOT NULL
)
SELECT *
FROM EmployeeList
ORDER BY Node
```

Now, the first section of the CTE is just metadata for setting up sample data for the Employee table.
Second section is made up of 2 parts, the first part called the anchor is getting the Boss (top dog, the one that has no boss i.e. President Trump). Second part is calling the rest of the employees and is a union of the inner join of the Employee and the named CTE section itself thus causing a recursion effect onto itself. Note I have also included the hierarchical level and the tree NodeID which gives us the hierachical tree path for any employee which in a sence its orgchart management structure and I can sought on this NodeID to get the data into the format I want.

### Solution 2 - hierachyid data type
The hierachyid was introduce for this very reason in that we don't have to write a recursion query anymore and we can achieve the same result. Instead of computing values recursively the type uses function method calls which is hard at first to understand without first having an understanding of programming concepts and actually seeing the t-sql code in practice, so you can depict a hierarchyid value as rather than a single value but a reference to a class that contains the value as binary and a function is used to retrieve the value in human readable form much the same as the NodeID in the previous solution. Without going into more details the reader can research the hierachyid data-type online just to note there are both static and class methods calls that enable inserting and querying data. Also it is possible to convert a table in the form used in solution 1 to that of solution 2 again search online for for information on MSDN.
