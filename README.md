![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Create SQL Database Mirroring Email Alerts
**Post Date: September 28, 2015**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>There are a number of ways to manually check, or manually configure some kind of alert method for Database Mirroring. The following is some SQL logic that will automatically configure SQL Database Mail AND send you an email alerts if your Database Mirroring status is anything but 'Synchronized'. 
  
  Here's what it does exactly.
1. Checks if Database Mail has been configured.
2. Configures Database Mail for you with a generic 'SQL Database Mail' profile.
3. Sends an automatic test email message after database mail has been configured showing you the server name, and instance from where the message originated.
4. Checks to see what databases have a status of anything other than 'Sychnorized'. 4. Creates a list of all databases that are not synchronized properly.
5. Captures the time difference ( number of minutes, hours, days ) since last synch.
6. Sends SMTP email with imbedded spreadsheet list formatted with HTML/CSS. This is NOT an attachment. This is directly in email.
All you need to do is replace this (MySMTPServerName.MyDomain.com) with your SMTP server name, and the @recipients line near the bottom. Make sure to change the 'SQLJobAlerts@MyDomain.com' to your email or distribution group. (preferably a distribution group) Then just throw it in a Job. Thats it.

Note:
This only runs on Database Servers with Database configured as the Principal.
This only reports the Databases that are presently configured with Database Mirroring, and those databases have fallen out of Synch.
This is safe to deploy on all Database Servers that have databases configured for Mirroring regardless of role. Principal or Mirror. You can create a job with this logic on either one. If it's simply setup as the Mirror; the Job will not send email unless a single database becomes the Principal. I use this on ALL my Database Servers that have Mirroring configured.
You're welcome :)</p>      


## SQL-Logic
```SQL
use msdb;
set nocount on
set ansi_nulls on
set quoted_identifier on
 
-- Configure SQL Database Mail if it's not already configured.
if (select top 1 name from msdb..sysmail_profile) is null
    begin
-- Enable SQL Database Mail
        exec master..sp_configure 'show advanced options',1
        reconfigure;
        exec master..sp_configure 'database mail xps',1
        reconfigure;
-- Add a profile
        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @description        = 'SQLDatabaseMail';
 
-- Add the account names you want to appear in the email message.
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @email_address      = 'sqldatabasemail@MyDomain.com'
        ,   @mailserver_name    = 'MySMTPServerName.MyDomain.com'  
        --, @port           = ####                  --optional
        --, @enable_ssl     = 1                 --optional
        --, @username       ='MySQLDatabaseMailProfile'         --optional
        --, @password       ='MyPassword'               --optional
 
        -- Adding the account to the profile
        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @sequence_number    = 1;
 
        -- Give access to new database mail profile (DatabaseMailUserRole)
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @principal_id       = 0
        ,   @is_default         = 1;
 
-- Get Server info for test message
 
        declare @get_basic_server_name              varchar(255)
        declare @get_basic_server_name_and_instance_name    varchar(255)
        declare @basic_test_subject_message         varchar(255)
        declare @basic_test_body_message            varchar(max)
        set @get_basic_server_name              = (select cast(serverproperty('servername') as varchar(255)))
        set @get_basic_server_name_and_instance_name    = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
        set @basic_test_subject_message         = 'Test SMTP email from SQL Server: ' + @get_basic_server_name_and_instance_name
        set @basic_test_body_message            = 'This is a test SMTP email from SQL Server:  ' + @get_basic_server_name_and_instance_name + char(10) + char(10) + 'If you see this.  It''s working perfectly :)'
 
-- Send quick email to confirm email is properly working.
 
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name   = 'SQLDatabaseMailProfile'
        ,   @recipients     = 'SQLJobAlerts@MyDomain.com'
        ,   @subject        = @basic_test_subject_message
        ,   @body           = @basic_test_body_message;
 
        -- Confirm message send
        -- select * from msdb..sysmail_allitems
    end
 
-- get basic server info.
 
declare @server_name_basic      varchar(255)
declare @server_name_instance_name  varchar(255)
set @server_name_basic      = (select cast(serverproperty('servername') as varchar(255)))
set @server_name_instance_name  = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
 
-- get basic server mirror role info.
 
declare
    @primary    varchar(255) = ( select @@servername )
,   @secondary  varchar(255) = ( select top 1 replace(left(mirroring_partner_name, charindex('.', mirroring_partner_name) - 1), 'TCP://', '') from master.sys.database_mirroring where mirroring_guid is not null )
,   @instance   varchar(255) = ( select top 1 mirroring_partner_instance from master.sys.database_mirroring where mirroring_guid is not null )
,   @witness    varchar(255) = ( select top 1 case mirroring_witness_name when '' then 'None configured' end from master.sys.database_mirroring where mirroring_guid is not null )
 
-- set message subject.
declare @message_subject    varchar(255)
set @message_subject    = 'Mirror Synchronization latency found on Server:  ' + @server_name_instance_name
 
-- create table to hold mirror synchronization information
 
if object_id('tempdb..#check_mirror_synch') is not null
    drop table #check_mirror_synch
 
create table #check_mirror_synch
(
    [database_name] varchar(255)
,   [mirror_status] varchar(255)
,   [time]      datetime
)
 
-- populate #check_mirror_synch table
 
insert into #check_mirror_synch
select 
    'database_name'     = upper(sd.name)
,   'mirror_status'     = 
    case mdbm.status 
        when 0 then 'SUSPENDED'
        when 1 then 'DISCONNECTED'
        when 2 then 'SYNCHRONIZING'
        when 3 then 'PENDING FAILOVER'
        when 4 then 'SYNCHRONIZED'
        else ''
    end
,   local_time
 
from
    master.sys.databases sd join msdb.dbo.dbm_monitor_data mdbm on sd.database_id = mdbm.database_id
    and mdbm.local_time = 
    (
        select  max(sub_mdbm.local_time) 
        from    msdb.dbo.dbm_monitor_data sub_mdbm
        where   mdbm.database_id = sub_mdbm.database_id and sub_mdbm.local_time <= getdate()
    )
group by
    sd.name
,   mdbm.database_id
,   mdbm.status
,   mdbm.local_time
order by
    sd.name
,   local_time
 
-- create table #check_mirror_latency
 
if object_id('tempdb..#check_mirror_latency') is not null
    drop table #check_mirror_latency
 
create table #check_mirror_latency
(
    [server_name]   varchar(255)
,   [database_name] varchar(255)
,   [mirror_status] varchar(255)
,   [last_synch]    datetime
,   [delta]     varchar(255)
)
 
-- populate table #check_mirror_latency
 
if exists(select top 1 * from #check_mirror_synch where [mirror_status] not in ('SYNCHRONIZED'))
    begin
        insert into #check_mirror_latency
        select
            [server_name]   = @@servername
        ,   [database_name]
        ,   [mirror_status]
        ,   [last_synch]    = left([time], 19) + ' ' + datename(dw, [time])
        ,   [delta]         = 
                            case
                                when datediff(second, [time], getdate()) / 60 / 60 / 24 / 30 / 12 = 0 then ''
                                    else cast(datediff(second, [time], getdate()) / 60 / 60 / 24 / 30 / 12 as nvarchar(50)) + ' Yr '
                            end + 
                            case
                                when datediff(second, [time], getdate()) / 60 / 60 / 24 / 30 % 12 = 0 then ''
                                    else cast(datediff(second, [time], getdate()) / 60 / 60 / 24 / 30 % 12 as nvarchar(50)) + ' Mo '
                            end +
                            case    
                                when datediff(second, [time], getdate()) / 60 / 60 / 24 % 30 = 0 then ''
                                    else cast(datediff(second, [time], getdate()) / 60 / 60 / 24 % 30 as nvarchar(50)) + ' Dy '
                            end +
                            case
                                when datediff(second, [time], getdate()) / 60 / 60 % 24 = 0 then ''
                                    else cast(datediff(second, [time], getdate()) / 60 / 60 % 24 as nvarchar(50)) + ' Hr '
                            end +
                            case
                                when datediff(second, [time], getdate()) / 60 % 60 = 0 then ''
                                    else cast(datediff(second, [time], getdate()) / 60 % 60 as nvarchar(50)) + ' mn '
                            end
                            -- seconds if required:
                            --+
                            --case
                            --  when datediff(second, @last_date, getdate()) % 60 = 0 then ''
                            --      else cast(datediff(second, @last_date, getdate()) % 60 as nvarchar(50)) + ' s'
                            --end
        from
            #check_mirror_synch
        where
            [mirror_status] not in ('SYNCHRONIZED')
    end
-- create conditions for html tables in top and mid sections of email.
 
declare @xml_top            NVARCHAR(MAX)
declare @xml_mid            NVARCHAR(MAX)
declare @body_top           NVARCHAR(MAX)
declare @body_mid           NVARCHAR(MAX)
 
-- set xml top table td's
-- create html table object for: #check_mirror_latency
 
set @xml_top = 
    cast(
        (select
            [server_name]   as 'td'
        ,   ''
        ,   [database_name] as 'td'
        ,   ''
        ,   [mirror_status] as 'td'
        ,   ''
        ,   [last_synch]    as 'td'
        ,   ''
        ,   [delta]     as 'td'
        ,   ''
 
        from  #check_mirror_latency
        order by [database_name] asc
        for xml path('tr')
        ,   elements)
        as NVARCHAR(MAX)
        )
 
-- set xml mid table td's
-- create html table object for: #extra_table_formatting_if_needed
/*
set @xml_mid = 
    cast(
        (select 
            [Column1]   as 'td'
        ,   ''
        ,   [Column2]   as 'td'
        ,   ''
        ,   [Column3]   as 'td'
        ,   ''
        ,   [...]       as 'td'
 
        from  #get_last_known_backups 
        order by [database], [time_of_backup] desc 
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )
*/
-- format email
set @body_top =
        '<html>
        <head>
            <style>
                    h1{
                        font-family: sans-serif;
                        font-size: 90%;
                    }
                    h3{
                        font-family: sans-serif;
                        color: black;
                    }
 
                    table, td, tr, th {
                        font-family: sans-serif;
                        border: 1px solid black;
                        border-collapse: collapse;
                    }
                    th {
                        text-align: left;
                        background-color: gray;
                        color: white;
                        padding: 5px;
                    }
 
                    td {
                        padding: 5px;
                    }
            </style>
        </head>
        <body>
        <H3>' + @message_subject + '</H3>
 
        <h1>
        <p>
        Primary Server:  <font color="blue">' + @primary      + ' </font><br>
        Secondary Server:  <font color="blue">' + @secondary  +
                        case
                            when @secondary <> @instance then @secondary + '\' + @instance
                            else ''
                        end + '                                 </font><br>
                 
        Witness Server:  <font color="blue>"' + @witness      + ' </font></p>
        </h1>
          
        <h1>Note: Last Mirror Synch Info</h1>
        <table border = 1>
        <tr>
            <th> Server Name  </th>
            <th> Database Name    </th>
            <th> Mirror Status    </th>
            <th> Last Synch       </th>
            <th> Delta            </th>
        </tr>'
 
set @body_top = @body_top + @xml_top +
/* '</table>
<h1>Mid Table Title Here</h1>
<table border = 1>
        <tr>
            <th> Column1 Here </th>
            <th> Column2 Here </th>
            <th> Column3 Here </th>
            <th> ...          </th>
        </tr>'        
+ @xml_mid */ 
'</table>
        <h1>Go to the server using Start-Run, or (Win + R) and type in: mstsc -v:' + @server_name_basic + '</h1>'
+ '</body></html>'
 
-- send email.
 
if exists(select top 1 * from master.sys.database_mirroring where mirroring_role_desc in ('PRINCIPAL'))
    begin
        if exists(select top 1 * from #check_mirror_synch where [mirror_status] not in ('SYNCHRONIZED'))
            begin
                exec msdb.dbo.sp_send_dbmail
                    @profile_name       = 'SQLDatabaseMailProfile'
                ,   @recipients     = 'SQLJobAlerts@MyDomain.com'
                ,   @subject        = @message_subject
                ,   @body           = @body_top
                ,   @body_format        = 'HTML';
            end
    end
 
drop table #check_mirror_synch
drop table #check_mirror_latency

```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

      
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

