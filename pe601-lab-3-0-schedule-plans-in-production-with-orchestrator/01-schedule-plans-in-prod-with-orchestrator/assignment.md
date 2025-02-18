---
slug: schedule-plans-in-prod-with-orchestrator
id: sf7hpmojacjx
type: challenge
title: Schedule Plans in Production with Orchestrator
notes:
- type: text
  contents: PLACEHOLDER
tabs:
- title: PE Console
  type: service
  hostname: puppet
  port: 443
- title: Primary Server
  type: terminal
  hostname: puppet
- title: Windows Workstation
  type: service
  hostname: guac
  path: /#/client/c/workstation?username=instruqt&password=Passw0rd!
  port: 8080
- title: Windows Agent
  type: service
  hostname: guac2
  path: /#/client/c/winagent1?username=instruqt&password=Passw0rd!
  port: 8080
- title: Linux Agent 1
  type: terminal
  hostname: nixagent1
- title: Linux Agent 2
  type: terminal
  hostname: nixagent2
- title: Git Server
  type: service
  hostname: gitea
  path: /
  port: 3000
difficulty: basic
timelimit: 7200
---
Use PQL to identify development nodes from the console
========

1. On the **PE Console** tab, login with the username **admin** and password **puppetlabs**
2. Click the **Nodes** page link
3. Change the **Filter by** dropdown to **PQL query** and select the value **Nodes assigned to a specific environment (example: production)**
4. Change the actual query text to read:
    ```
    nodes[certname] { catalog_environment = "development" }
    ```
5. Run the query and copy the results to a local text editor for use in the next section

Test a Puppet plan against development nodes using the console
========

1. On the **PE Console** tab, click the **Plans** page link on the left hand navigation menu
2. Click the blue **Run a plan** button in the top right corner of the Plans page
3. Click the **Code environment** dropdown and select **development**
4.  Click the **Plan** dropdown and select **nginx::backup_all_logs**
5. In the Parameter section, add agent node names gathered in the first section to the **Value** field of the **targets** parameter using a comma separator.
6. Click the **Run job** button
7. Observe that the plan completes successfully.

Verify the backup directories have been created on the development nodes
========

1. Switch to the **Windows Agent** tab, and click the **Windows Explorer** icon in the system tray. Click **This PC**, then double-click **Local Disk (C:)** > **backups**.
2. Double-click the date-stamped directory and observe the nginx log files that have been backed up.
3. Switch to the **Linux Agent 2** tab, and run the following command:
    ```
    cd /var/backups
    ```
4. `cd` into the date-stamped directory and observe the nginx log files that have been backed up.

Create a service account to secure running the backup plan in production
========

1. Switch back to the **PE Console** tab
2. Click the **Access Control** link on the left hand navigation menu
3. Click the **User Roles** tab and create a new role by entering the text `plan_automation_service` in the *name* field on the first empty row
4. Click the **Add Role** button next to the role name you just entered
5. Click the **plan_automation_service** row that now appears in the list
6. Click the **Permissions** tab
7. Add the following permissions to the role, clicking the **Add** button after each selection:
   ```
   Plans - Run Plans - nginx::backup_all_logs
   Job Orchestrator - Start, stop, and view jobs
   ```
8. Click the **Commit 2 changes** button
9. Click the **Access Control** link from the left hand nav menu again
10. Add a new user account with Full name: `plan_automation_service_account` and Login: `plan_automation_service_account`
11. Click the **Add local user** button next to the new row
12. Click the **User roles** tab, click the `plan_automation_service` link and add the `plan_automation_service_account` to the `plan_automation_service` role
13. Click **Commit 1 change** button
14. Click the `plan_automation_service_account` link and click the **Generate password reset** link
15. Copy the link into a new browser tab, enter the password **puppetlabs**, and click **Reset password**
16. Close the browser tab and click the **Close** link in the PE console

Use PQL and an access token to schedule a plan run for production with the Orchestrator API
========

1. Switch to the **Primary Server** tab
2. Run the following command to create an access token for your service account user credentials entering the **puppetlabs** password configured in the last step when prompted:
    ```
    puppet access login plan_automation_service_account -t /root/.puppetlabs/token.plan-svc-acct
    ```
3. Create a variable to hold the authentication header needed to make the API call by by running the following in the termnial and copying and pasting your token value where specified:
    ```
    header="X-Authentication: $(cat /root/.puppetlabs/token.plan-svc-acct)"
    ```

4. Use PQL to find all of the non-PE server nodes in the production environment:
    ```
    puppet query 'nodes[certname] { catalog_environment = "production" and certname !~ "puppet." }'
    ```

5. Using a PQL query and the `jq` utility, create a variable to hold the list of production nodes:
    ```
    node_list=$(puppet query 'nodes[certname] { catalog_environment = "production" and certname !~ "puppet." }' | jq 'map(.certname)  | join(",")')
    ```

6. Create a date variable with a timestamp 5 minutes from now:
    ```
    run_date=$(date -d '5 minutes' +"%Y-%m-%dT%H:%M:%S%z")
    ```

7. Run the following to schedule your plan run using curl and the variables you previously created:
    ```
    curl -X POST -k -H "$header" https://localhost:8143/orchestrator/v1/command/schedule_plan \
    --data-binary @- << EOF
    {
    "environment": "production",
    "plan": "nginx::backup_all_logs",
      "params": {
        "targets": $node_list
      },
      "scheduled_time": "$run_date"
    }
    EOF
    ```

8. Switch to the **PE Console** and click the **Plans** link
9. Click the **Scheduled Plans** link to view your scheduled plan
10. Wait 5 minutes until the scheduled plan runs and disappears from the scheduled plans list

Verify the backup directories have been created on the production node
========

1. Switch to the **Linux Agent 1** tab, and run the following command:
    ```
    cd /var/backups
    ```
2. `cd` into the date-stamped directory and observe the nginx log files that have been backed up.

🎈 **Congratulations!** In this lab, you tested your Puppet plan to backup nginx log files on nodes in the development environment. Then you configured role-based access control (RBAC) for a service account user with permissions to run the plan on production agent nodes. Finally, you configured a scheduled job using the `schedule_plan` API endpoint to run the backup logs plan on your production Linux node and verified that it completed successfully.