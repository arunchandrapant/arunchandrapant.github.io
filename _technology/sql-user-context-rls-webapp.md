---
classes: "wide"
title: "Connecting to Azure SQL DB using access token obtained on behalf of users, from a web application, to take advantage of row level security thereby offloading data access-control logic from web app to DB."
excerpt: "Azure SQL DB accepts access token generated on behalf of user, to run queries. This combined with row level security, one can offload access control logic on data completely from web app to SQL server"
---
## What is the Use Case?
In a web app, what data is exposed to end users is generally controlled at web backend level. In such cases, when user requests data from front end, the corresponding SQL query (assuming data has to come from SQL server) is generated on backend. Application adds appropriate checks and balances to the query to ensure that only data belonging to that user is fetched. In case extra data is fetched due to some need, it will be filtered by application, before it is displayed to front end. Coding this querying and filtering logic can become very complicated in some cases. Also it must be developed very carefully and tested properly as poor quality code can lead to data breach.

One alternative is to offload this logic from application level to database level. This can be done by combining largely used table and column level security with ['row level security'](https://docs.microsoft.com/en-us/sql/relational-databases/security/row-level-security?view=sql-server-2017) feature present in Azure SQL DB (and other SQL server versions). Together they offer complete control on what data a user can get from database. Yes this would mean creating all the users in the DB and giving them direct access to the DB. But in scenarios where this is already the case, it won't be much overhead or security concern. Data warehouse is one such case where typically direct access to DB is given to BI developers to connect and create BI reports. When number of users are large (such as an entire organization), such access can be given by [creating security groups in Azure Active Directory and assigning them to roles in DB](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-manage-logins#groups-and-roles), which is also the recommended way to assign permissions on Azure SQL DB. Also in cases where many applications are accessing same DB with same permissions, managing access control directly from DB can make sense.

However, configuring row level security is only the first step. You need a way for web app to run queries on DB by impersonating the user. Here, a feature of Azure SQL DB which lets you [authenticate using Azure AD access token](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-aad-authentication-configure#azure-ad-token) comes into play. In this scenario, user needs to authenticate themselves to the application using Azure AD authentication on the front end. Application can then obtain an access token on behalf of user to access Azure SQL DB by impersonating his identity. When application queries DB using this access token, only the rows accessible to the user as defined in row level security are returned. Hence application can just assume that user has access to entire rows in tables where he has access and issue same query for a particular requirement for every user. The DB will transparently filter rows and return only those rows where user has legitimate access as defined in DB. Combining this with appropriate column and table level access, we can cover all possible access control scenarios.

But in my view this is not a popular way of doing things and can lead to many issues. Particularly, every query will come with an attached overhead of checking row level access. On application this can be handled fast, but on DB this leads to many joins (as row level security is checked thorugh a table valued inline function attached to a security policy) which can degrade query performance. Also many applications might not be comfortable with DB schema being exposed to end users. I have no idea of scalability when it comes to giving access to large number of users or to a very large amount of data. What about applications with millions of users? Does row level security create [compatibility issues](https://docs.microsoft.com/en-us/sql/relational-databases/security/row-level-security?view=sql-server-2017#Limitations) with other features in DB? However, in case of our application all these issues are not present as of now. So, we can give it a try and see how it goes about.

In the following sections, I will give a technical proof of concept of how this will work in a web application. Since, I am not a web developer, I will simulate front and backend scenarios using frugal methods. However, any web developer will easily translate it into his application.

This article assumes basic understanding of how authentication and authorization through Azure AD works.

## Configure Your Web App in Azure AD
[Register](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app) a [web app](https://docs.microsoft.com/en-us/azure/active-directory/develop/web-app) in Azure AD. [Give this app permissions](https://docs.microsoft.com/en-us/cli/azure/ad/app/permission?view=azure-cli-latest#az-ad-app-permission-add) to sign in users and access Azure SQL DB and Warehouse on behalf of signed in user. This permission is a [delegated permission](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent#permission-types) and not an application permission. Hence, when user signs in using this web app, he will be prompted to allow application to access DB. If a user responds in negative, application will not be able to run queries by impersonating him.

If you need to give your app permission to Azure SQL DB API in Azure AD using Azure CLI, you will need API ID. This can be very hard to find as they have [not been documented](https://github.com/Azure/azure-cli/issues/7925#issuecomment-511543237) anywhere by Microsoft. They recommend adding it to an app from Azure AD portal and copying it from it's manifest file. But in organization like mine where users are not allowed to access Azure AD web portal, but can access Azure AD through Azure CLI, this will not work. I hope that some day Microsoft will document these codes or that my organization will allow me to access Azure AD web portal. For the meanwhile, following are the codes (that were extracted with pain):

Code for Azure AD `Azure SQL Database` API:
> 022907d3-0f1b-48f7-badc-1ba6abab6d66

Code for `Access Azure SQL DB and Data Warehouse` delegated permission of above API:
> c39ef2d1-04ce-46dc-8b5f-e9a5c60f0fc9

Azure CLI command for assigning above permission to the app (app id '--id' parameter is dummy here):
> az ad app permission add --id 6f5163d2-b31d-4404-92e1-a22564dac425 --api 022907d3-0f1b-48f7-badc-1ba6abab6d66 --api-permissions c39ef2d1-04ce-46dc-8b5f-e9a5c60f0fc9=Scope

## The Frontend Part
These things happen on user facing side of the application. Redirect user to following URL for authentication:

> https://login.microsoftonline.com/f769f446-cc0d-478d-86b9-6b8c9a58c4a4/oauth2/authorize?client_id=6f5163d2-b31d-4404-92e1-a22564dac425&response_type=code&redirect_uri=https%3A%2F%2Fyouappurlhere&response_mode=query&resource=https%3A%2F%2Fdatabase.windows.net%2F

See [this link](https://docs.microsoft.com/en-us/azure/active-directory/develop/v1-protocols-oauth-code#request-an-authorization-code) to understand above request in more detail.

Following image shows how permission dialogue box will appear to user when they go to above link (some permissions are shown extra here):

<figure class="align-center">
  <a href="/assets/images/technology/app_login.JPG">
  <img src="/assets/images/technology/app_login.JPG" alt="" width="100" height="100" style="object-fit: cover">
  </a>
  <figcaption>This is how permissions show up when logging in. ("Access azure service management as you" permission shown is extra, and need not be present for this case.)</figcaption>
</figure>

After user successfully authenticates himself using his Azure AD username and password and allows app to run SQL queries on their behalf, response containing code will be send to `redirect_uri` mentioned in above URL. The response URL will look as follows:

> https://yourappurlhere/?code=AQABAAIAAAAP0wveryverylongcodehere17cyIgdWWhAjTIgAA&session_state=f88afa06-35f6-4992-a5b1-c8efe735faf2

See [this link](https://docs.microsoft.com/en-us/azure/active-directory/develop/v1-protocols-oauth-code#successful-response) to understand response in detail.

## The Backend Part

Catch the code from above `redirect_uri` in your web app. Use this code in following POST request to get access token. I am using Python to send this post request (emulating web backend). To understand request made in following code see [this link](https://docs.microsoft.com/en-us/azure/active-directory/develop/v1-protocols-oauth-code#use-the-authorization-code-to-request-an-access-token).

<script src="https://gist.github.com/arunchandrapant/4dd2ce06a979915613e3a35494831fe9.js"></script>

The response will contain access token, refresh token and id token. [This link](https://docs.microsoft.com/en-us/azure/active-directory/develop/v1-protocols-oauth-code#successful-response-1) explains the response you get in detail. We require access token to make requests to Azure SQL DB.

### Now Get the Data

To setup connection to Azure SQL DB and send queries to it, we will need a driver. Here I have used ODBC 13 driver for SQL Server. Previous versions of ODBC driver for SQL Server do not support access token based authentication. To access this driver from python use [pyodbc package](https://github.com/mkleehammer/pyodbc).

The python code to run SQL queries using access code is below.

<script src="https://gist.github.com/arunchandrapant/95b230363a6c3a7d1d6147b037bb9deb.js"></script>

You will see a section marked as `BLACK MAGIC STARTS` in the comments in above python script. This is a bit complicated part. It breaks access token and add parts to a struct object, which is required to set connection attributes before creating connection. The C code for the same is documented [here](https://docs.microsoft.com/en-us/sql/connect/odbc/using-azure-active-directory?view=sql-server-2017#azure-active-directory-authentication-sample-code). I don't know why would SQL Server provide such a complicated way of exposing functionality to user. Some [explanation is provided here](https://github.com/mkleehammer/pyodbc/issues/228#issuecomment-318414803). This code stunt should have been hidden inside some library function. Then end users would expect that they simply pass access token to a function that creates connection. However, when using python, things get more complicated. This is because C datatypes cannot be directly manipulated in Python. So some code magic doing the same is implemented [here](https://github.com/mkleehammer/pyodbc/issues/228#issuecomment-318414803). If you can understand this, congratulations. I can't as of now, so I have just copy-pasted in the code above.

So finally we have put all the parts in place and successfully fetched data from Azure SQL DB using access token on behalf of users to take advantage of row level security for effective data control.

However, above steps are only for the first time a user signs in. The next time user logs in, we have to follow a slightly different path. This path uses [refresh token](https://docs.microsoft.com/en-us/azure/active-directory/develop/v1-protocols-oauth-code#refreshing-the-access-tokens) to obtain access token as opposed to auth code required during first login. For this purpose, refresh token generated during first login will have to be cached by application.

If you need to access Azure SQL DB using other platforms such as .NET, Powershell etc. you would like to check links [here](https://docs.microsoft.com/en-us/sql/connect/oledb/features/using-azure-active-directory?view=sql-server-2017#azure-active-directory-authentication-using-an-access-token
), [here](https://techcommunity.microsoft.com/t5/Azure-Database-Support-Blog/Azure-SQL-Database-Token-based-authentication-with-PowerShell/ba-p/369078), [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-aad-authentication-configure#azure-ad-token), [here](https://michaelcollier.wordpress.com/2016/11/03/connect-to-azure-sql-database-by-using-azure-ad-authentication/#add-sqldb-permission) and [here](https://winterdom.com/2017/08/31/token-delegation-azure-sql). Each of these links might give you hints but might not completely fit into your situation.
