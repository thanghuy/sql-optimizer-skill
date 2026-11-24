---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

SQL Server 2019 (compat 150). Our "active buyers" search is slow and CPU spikes when support uses it. Users ~2M rows, Orders ~40M rows. What should we fix?

Users(UserId int PK clustered, UserCode varchar(20) NOT NULL, UserName nvarchar(100), Email nvarchar(200), IsDeleted bit NULL)
  -- indexes: IX_Users_UserCode (UserCode), IX_Users_UserName (UserName)
Orders(OrderId bigint PK clustered, UserCode nvarchar(20) NOT NULL, UserId int NOT NULL, OrderDate datetime2 NOT NULL, Amount money)
  -- indexes: IX_Orders_UserCode (UserCode), IX_Orders_OrderDate (OrderDate)

Main query (called from C# with @search as nvarchar):
SELECT DISTINCT u.UserId, u.UserName, u.Email
FROM dbo.Users u
JOIN dbo.Orders o ON o.UserCode = u.UserCode
WHERE ISNULL(u.IsDeleted, 0) = 0
  AND YEAR(o.OrderDate) = 2025
  AND u.UserName LIKE @search + '%';

And before opening a user's detail page the app runs:
IF (SELECT COUNT(*) FROM dbo.Orders WHERE UserId = @id) > 0
    SELECT ... -- load page
