---
title: Azure SQL Edge usage and diagnostics data configuration
description: Learn how to configure usage and diagnostics data in Azure SQL Edge.
services: sql-edge
ms.service: sql-edge
ms.topic: conceptual
author: rothja
ms.author: jroth
ms.reviewer: jroth
ms.date: 08/04/2020
---

# Azure SQL Edge usage and diagnostics data configuration

By default, Azure SQL Edge collects information about how its customers are using the application. Specifically, Azure SQL Edge collects information about the deployment experience, usage, and performance. This information helps Microsoft improve the product to better meet customer needs. For example, Microsoft collects information about what kinds of error codes customers encounter so that we can fix related bugs, improve our documentation about how to use Azure SQL Edge, and determine whether features should be added to the product to better serve customers.

Specifically, Microsoft does not send any of the following types of information through this mechanism:

- Any values from inside user tables.
- Any logon credentials or other authentication information.
- Any personal or customer data.

The following sample scenario includes feature usage information that helps improve the product.

An example query from the queries used for the usage and diagnostics data collection is provided below. The query identifies the count and types of different streaming data sources being used in Azure SQL Edge. This data helps Microsoft identify which streaming data sources are being used commonly such that Microsoft can improve the performance and user experience associated with these data sources. 

```sql
select 
count(*) as [count], sum(inputs) as inputs, sum(outputs) as outputs, sum(linked_to_job) 
as linked_to_job, data_source_type
from ( 
select isnull(value,'unknown') as data_source_type, inputs, outputs, linked_to_job
from 
    ( 
        select 
        convert(sysname, lower(substring(ds.location, 0, charindex('://', ds.location))), 1) as data_source_type, 
        isnull(inputs, 0) as inputs, isnull(outputs, 0) as outputs, isnull(js.stream_id/js.stream_id, 0) as linked_to_job 
        from sys.external_streams es
        join sys.external_data_sources ds 
             on es.data_source_id = ds.data_source_id
        left join 
            ( 
            select stream_id, max(cast(is_input as int)) inputs, max(cast(is_output as int)) outputs 
            from sys.external_job_streams group by stream_id 
            ) js
             on js.stream_id = es.object_id 
    ) ds
left join 
    (
        select value from string_split('edgehub,sqlserver,kafka', ',')) as known_ep on data_source_type = value 
    ) known_ds
group by data_source_type
```

## Disable usage and diagnostic data collection

Usage and diagnostic data collection on Azure SQL Edge can be disabled using either of the below methods.

> [!NOTE]
> Usage and diagnostic data cannot be disabled for the Developer version.

### Disable usage and diagnostics using environment variables

To disable Usage and Diagnostics data collection on Azure SQL Edge, add the following environment variable and set its value to `*False*`. For more information on configuring Azure SQL Edge using environment variables, refer [Configure using Environment Variables](configure.md#configure-by-using-environment-variables).

`MSSQL_TELEMETRY_ENABLED = TRUE | FALSE`

- TRUE - Enables collection of usage and diagnostics data. This is the default configuration.
- FALSE - Disables collection of usage and diagnostics data.

### Disable usage and diagnostics using mssql.conf file

To disable Usage and Diagnostics data collection on Azure SQL Edge, add the following lines in the mssql.conf file on the persistent storage drive that is mapped to the /var/opt/mssql/ folder in the SQL Edge module. For more information on configuring Azure SQL Edge using mssql.conf file, refer [Configure using mssql.conf file](configure.md#configure-by-using-an-mssqlconf-file).

```ini
[telemetry]
customerfeedback = false
```

## Local audit of usage and diagnostic data collection

The Local Audit component of Azure SQL Edge Usage and Diagnostic Data collection can write data collected by the service to a designated folder, representing the data (logs) that will be sent to Microsoft. The purpose of the Local Audit is to allow customers to see all data Microsoft collects with this feature, for compliance, regulatory or privacy validation reasons.

### Enable local audit of usage and diagnostics data

To enable Local Audit usage and diagnostics data on Azure SQL Edge

1. Create a target directory for new Local Audit log storage. This target directory can either be on the host or within the container. In the example below, target directory is created in the same mount volume that is mapped to /var/opt/mssql/ path on SQL Edge.

   ```bash
   sudo mkdir <host mount path>/audit
   ```

2. Configure audit of usage and diagnostics data using either environment variables or mssql.conf file.

   - Using environment variables - Add the following environment variable to your SQL Edge deployment and specify the target directory for the audit files.
   
     `*MSSQL_TELEMETRY_DIR = <host mount path>/audit*`
   
   - Using mssql.conf file - Add the following lines in the mssql.conf file and specify the target directory for the audit files.
       ```ini
       [telemetry]
       userrequestedlocalauditdirectory  = <host mount path>/audit
       ```  

## Next steps

- [Connect to Azure SQL Edge](connect.md)
- [Build an end-to-end IoT solution with SQL Edge](tutorial-deploy-azure-resources.md)
