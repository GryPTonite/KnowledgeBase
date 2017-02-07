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
There is no specific event client side so you have to handle your exceptions in every function however returning code from teh server in an ajax call you can make use of callback onerror event handler to notify the user.

### SQL
5. Joining on non mandatory master/child 1 to many relationship tables to display only rows where there is no child relationship

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
Here by using a join between the two tables the process is aligant and most effecient - we can assume (best practice is the ID column is the primary key and thus the clustered index) that both tables are pre-ordered, so then the first action is to just sort the child table by Master ID and only have distinct values, so then all the process requires is one traverse of the master and child tables together as supposed to many look-ups into the child table that the first solution suggests.


