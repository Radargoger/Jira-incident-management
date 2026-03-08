# Incident Management for Jira - Configuration Guide

Welcome to the SOC Platform Incident Management app! This integration empowers your SOC team by seamlessly syncing threat intelligence alarms from SOC Platform directly into your Jira workflows.

## How It Works (Core Principles)
* **Automated Polling:** Once enabled, the application automatically checks for new SOC Platform alarms **every 5 minutes**. 
* **Safe Formatting:** Threat intelligence data can be massive. To ensure Jira runs smoothly, the app automatically formats data (like IP addresses, domains, and compliance lists) into clean tables. If a description is extremely long, it safely truncates the text and provides a note to view the full details on the SOCRadar platform.
* **Bi-Directional Sync:** When an alarm's status changes in SOCRadar, the corresponding Jira issue is updated. Conversely, if your team closes the issue in Jira, the app updates the status in SOC Platform (based on your mapping rules).

## Setting Up the Integration
After installing the app, navigate to **Apps > SOC Platform Integration** in your Jira top navigation menu to access the Configuration Dashboard.

### 1. Connection Tab (Core Setup)
This is where you establish the secure connection between Jira and your SOC Platform tenant.

* ** API:**
  * **Company ID & API Key:** Enter your credentials provided by SOC Platform. 
  * **User Email:** Enter the email address of the user who is authorizing this integration.
* **Jira Project:**
  * **Project Key:** Enter the exact Key of the Jira project where you want SOC Platform tickets to be created (e.g., `SEC`, `IT`, `SOC`).
  * **Issue Type:** Select whether the alarms should be created as a `Task` or an `Epic`.
* **Sync Options:**
  * **Enable automatic sync:** Turn this ON to start the 5-minute polling cycle.
  * **Auto-create Jira issues:** If enabled, new alarms will automatically generate new Jira tickets.
  * **Sync comments:** If enabled, status transition notes are sent as comments between platforms.

*Don't forget to click **Test Connection** to ensure your credentials are correct, then click **Save Configuration**.*

### 2. Filters Tab
Control exactly which alarms are sent to your Jira project to avoid alert fatigue.

* **Status & Severity:** Filter alarms by their current SOC Platform status (e.g., OPEN) or Severity (e.g., CRITICAL, HIGH).
* **Alarm Main Types:** Click on the specific threat categories you want to monitor (e.g., Brand Protection, Dark Web Monitoring, Vulnerability Intelligence).
* **Tags & Assignees:** Narrow down incoming alerts by entering specific comma-separated tags or assigned users.

### 3. Mapping Tab (Crucial for Bi-Directional Sync)
To keep both platforms perfectly aligned, you must map SOC Platform statuses to your specific Jira workflow statuses.

* **SOC Platform → Jira:** Define what happens in Jira when an SOC Platform alarm changes status. (e.g., When  Platform says "RESOLVED", transition Jira to "Done").
* **Jira → SOC Platform:** Define what happens in SOC Platform when your team moves a Jira ticket. (e.g., When a Jira ticket is moved to "In Progress", transition SOC Platform to "INVESTIGATING").
* **Add Custom Jira Status:** If your Jira project uses custom status names (like "Under Review" or "Awaiting Vendor"), use the `+ Add` button at the bottom to add them to the dropdown lists.

### 4. Logs Tab
Use this tab for troubleshooting. It provides real-time logs of the synchronization process. You can see exactly when the last poll occurred, how many alarms were fetched, and if any errors (like a missing Jira status in your workflow) prevented a ticket from being created.
