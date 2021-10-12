---
title: "Contained Database Users in SQL"
summary: "What is a contained database user, and how to use them?"
date: "2020-06-15T19:02:00+10:00"
lastmod: "2020-06-15T19:02:00+10:00"
author: Gary Jackson
draft: false
categories:
- "Development"
tags:
- "SQL"
- "Azure"
- "Azure SQL"
---

## Overview

Contained database users are users that are configured directly at the database level, and don't require an associated login in the master database.

The benefit of this approach is that it makes your database more portable - and it also simplifies database deployments a little.

It looks like contained databases have been available for years now, but I've only just become aware of contained database users in the last couple months - so I figure, maybe I'm not the only one..

## Setup
If you have docker installed locally and would to play along at home, here is a quick docker-compose file to get you up and running with your own local SQL instance.

{{< admonition type=note title="Note" open=true >}}
This compose assumes your docker is configured for linux containers.
{{< /admonition >}}

```yaml
version: '3'
services:
  sql:
    image: "mcr.microsoft.com/mssql/server:2019-latest"
    hostname: sql2019
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=#YourSuperSecretPassword123
    ports:
      - "1433:1433"
```

Once the SQL instance is up and running, connect to the instance using the credentials in the docker-compose, and run the following script to enable contained database authentication.

```SQL
USE [master]

EXEC sp_configure 'CONTAINED DATABASE AUTHENTICATION', 1
RECONFIGURE

GO
```

## The Traditional Login + User Model
The traditional model for providing a user with access to a database was a two step process.

1. Create a `login` in the master database
2. Create a `user` in the target database mapped to the login from step 1.

The login is to provide `authentication` to the SQL `instance`, whereas the user provides `authorisation` to the SQL `database`.

Here is a sample SQL script you can run to see it in action.

```SQL
USE [master]
GO

CREATE DATABASE [NoneContainmentDatabase]
 CONTAINMENT = NONE
GO

CREATE LOGIN [MyTraditionalUserLogin] WITH PASSWORD = '#password1'
GO

USE [NoneContainmentDatabase]
GO

CREATE USER [MyTraditionalUser] FOR LOGIN [MyTraditionalUserLogin]
GO
```

Using SSMS, you should see something like this.

{{< figure src="images/TraditionalModel.PNG" alt="Traditional Model"  >}}

## The Contained Database User Model
In contrast, the contained database user model only needs the `CREATE USER`, with the addition of password parameter.

For example

```SQL
USE [master]

CREATE DATABASE [PartialContainmentDatabase]
 CONTAINMENT = PARTIAL
GO

USE [PartialContainmentDatabase]
GO

CREATE USER [MyContainedUser] WITH PASSWORD='#password1'
GO
```

Producing the following

{{< figure src="images/ContainedUserModel.PNG" alt="Contained Database User Model"  >}}

### Verify the User
Verify that you can connect to the database - make sure to set the database name in the connection properties or you will get an error.

{{< figure src="images/VerifyConnection.PNG" alt="Verify the user"  >}}

And success!
{{< figure src="images/VerifiedConnection.PNG" alt="Verified connection"  >}}


## References
- [Contained Database Users - Making Your Database Portable](https://docs.microsoft.com/en-au/sql/relational-databases/security/contained-database-users-making-your-database-portable?view=sql-server-ver15)