---
title: Connect data sources through Logstash to Azure Sentinel | Microsoft Docs
description: Learn how to use Logstash to forward logs from external data sources to Azure Sentinel.
services: sentinel
documentationcenter: na
author: yelevin
manager: rkarlin
editor: ''

ms.service: azure-sentinel
ms.subservice: azure-sentinel
ms.devlang: na
ms.topic: how-to
ms.custom: mvc
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 05/27/2020
ms.author: yelevin

---
# Use Logstash to connect data sources to Azure Sentinel

> [!IMPORTANT]
> Data ingestion using the Logstash output plugin is currently in public preview. This feature is provided without a service level agreement, and it's not recommended for production workloads. For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

Using Azure Sentinel's new output plugin for the **Logstash data collection engine**, you can now send any type of log you want through Logstash directly to your Log Analytics workspace in Azure Sentinel. Your logs will be sent to a custom table that you will define using the output plugin.

To learn more about working with the Logstash data collection engine, see [Getting started with Logstash](https://www.elastic.co/guide/en/logstash/current/getting-started-with-logstash.html).

## Overview

### Architecture and background

   ![Logstash architecture](./media/connect-logstash/logstash-architecture.png)

The Logstash engine is comprised of three components:

- Input plugins - customized collection of data from various sources
- Filter plugins - manipulation and normalization of data according to specified criteria
- Output plugins - customized sending of collected and processed data to various destinations

> [!NOTE]
> Azure Sentinel supports its own provided output plugin only. It does not support third-party output plugins for Azure Sentinel, or any other Logstash plugin of any type.

The Azure Sentinel output plugin for Logstash sends JSON-formatted data to your Log Analytics workspace, using the Log Analytics HTTP Data Collector REST API. The data is ingested into custom logs.

- [Learn more about the Log Analytics REST API](https://docs.microsoft.com/rest/api/loganalytics/create-request).
- [Learn more about custom logs](https://docs.microsoft.com/azure/azure-monitor/platform/data-sources-custom-logs).

## Installing and configuring the Azure Sentinel output plugin in Logstash

1. **Installation**

    The Azure Sentinel output plugin is available in the Logstash collection.

    - Follow the instructions in the Logstash [Working with plugins](https://www.elastic.co/guide/en/logstash/current/working-with-plugins.html) document to install the ***microsoft-logstash-output-azure-loganalytics*** plugin.
    - If your Logstash system does not have Internet access, follow the instructions in the Logstash [Offline Plugin Management](https://www.elastic.co/guide/en/logstash/current/offline-plugins.html) document to prepare and use an offline plugin pack. (This will require you to build another Logstash system with Internet access.)

1. **Configuration**

    Using the information in the Logstash [Structure of a config file](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html) document, add the Azure Sentinel output plugin to the configuration with following keys and values (see the proper config file syntax below the table):

    | **Field name** | **Data type** | **Description** |
    |----------------|---------------|-----------------|
    | `workspace_id` | string | Enter your workspace ID GUID |
    | `workspace_key` | string | Enter your workspace primary key GUID |
    | `custom_log_table_name` | string | Set the name of the table into which the logs will be ingested. Only one table name per output plugin can be configured. The log table will appear in Azure Sentinel under **Logs**, in **Tables** in the **Custom Logs** category, with a `_CL` suffix. |
    | `endpoint` | string | Optional field. By default, this is the Log Analytics endpoint. Use this field to set an alternative endpoint. |
    | `time_generated_field` | string | Optional field. This property overrides the default **TimeGenerated** field in Log Analytics. Enter the name of the timestamp field in the data source. The data in the field must conform to the ISO 8601 format (`YYYY-MM-DDThh:mm:ssZ`) |
    | `key_names` | array | Enter a list of Log Analytics output schema fields. Each list item should be enclosed in single quotes and the items separated by commas, and the entire list enclosed in square brackets. See example below. |
    | `plugin_flush_interval` | number | Optional field. Set to define the maximum interval (in seconds) between message transmissions to Log Analytics. The default is 5. |
    | `amount_resizing` | boolean | True/false. Enable or disable the automatic scaling mechanism, which adjusts the message buffer size according to the volume of log data received. |
    | `max_items` | number | Optional field. Applies only if `amount_resizing` set to "false." Use to set a cap on the message buffer size (in records). The default is 2000.  |

    **Sample configuration**

        input {
            tcp {
                port => 514
                type => syslog
            }
        }

        filter {
            grok {
                match => { "message" => "<%{NUMBER:PRI}>1 (?<TIME_TAG>[0-9]{4}-[0-9]{1,2}-[0-9]{1,2}T[0-9]{1,2}:[0-9]{1,2}:[0-9]{1,2})[^ ]* (?<HOSTNAME>[^ ]*) %{GREEDYDATA:MSG}" }
            }
        }

        output {
            logstash-output-azure-loganalytics {
                workspace_id => "<WS_ID>"
                workspace_key => "${WS_KEY}"
                custom_log_table_name => "logstashCustomTable"
                key_names => ['PRI','TIME_TAG','HOSTNAME','MSG']
                plugin_flush_interval => 5
            }
        } 

    > [!NOTE]
    > Visit the output plugin’s [GitHub](https://github.com/Azure/Azure-Sentinel/tree/master/DataConnectors/microsoft-logstash-output-azure-loganalytics) to learn more about its inner workings, configuration, and performance settings.

1. **Restart Logstash**

1. **View incoming logs in Azure Sentinel**

    1. Verify that messages are being sent to the output plugin.

    1. From the Azure Sentinel navigation menu, click **Logs**. Under the **Tables** heading, expand the **Custom Logs** category. Find and click the name of the table you specified (with a `_CL` suffix) in the configuration.

        ![Logstash custom logs](./media/connect-logstash/logstash-custom-logs-menu.png)

    1. To see records in the table, query the table, using the table name as the schema.

        ![Logstash custom logs query](./media/connect-logstash/logstash-custom-logs-query.png)

## Monitor output plugin audit logs

To monitor the connectivity and activity of the Azure Sentinel output plugin, enable the appropriate Logstash log file. See the [Logstash Directory Layout](https://www.elastic.co/guide/en/logstash/current/dir-layout.html#dir-layout) document for the log file’s location.

If you are not seeing any data in this log file, generate and send some events locally (through the input and filter plugins) to make sure the output plugin is receiving data. Azure Sentinel will support only issues relating to the output plugin.

## Next steps

In this document, you learned how to use Logstash to connect external data sources to Azure Sentinel. To learn more about Azure Sentinel, see the following articles:
- Learn how to [get visibility into your data, and potential threats](quickstart-get-visibility.md).
- Get started detecting threats with Azure Sentinel, using [built-in](tutorial-detect-threats-built-in.md) or [custom](tutorial-detect-threats-custom.md) rules.