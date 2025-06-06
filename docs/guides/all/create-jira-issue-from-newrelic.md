---
displayed_sidebar: null
description: Learn how to report a bug in Port, ensuring prompt issue resolution and improving overall platform reliability.
---

import GithubActionModificationHint from '/docs/guides/templates/github/_github_action_modification_required_hint.mdx'
import GithubDedicatedRepoHint from '/docs/guides/templates/github/_github_dedicated_workflows_repository_hint.mdx'
import ExistingSecretsCallout from '/docs/guides/templates/secrets/_existing_secrets_callout.mdx'
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Report a Jira bug from a New Relic alert

## Overview
This guide will help you implement a self-service action in Port that allows you to report bugs in Jira based on New Relic alert in one click, ensuring prompt issue resolution and improving overall platform reliability.

You can implement this action in two ways:
1. **Synced webhooks**: A simpler approach that directly interacts with Jira's API through Port, ideal for quick implementation and minimal setup.
2. **GitHub workflow**: A more flexible approach that allows for complex workflows and custom logic, suitable for teams that want to maintain their automation in Git.

## Prerequisites

- Complete the [onboarding process](/getting-started/overview).
- Access to your Jira organization with permissions to create issues.
- [Port's GitHub app](https://github.com/apps/getport-io) installed (required for GitHub Actions implementation).
- [Jira API token](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/) with permissions to create new issues.

## Set up data model

If you have installed Port's [Jira integration](https://docs.port.io/build-your-software-catalog/sync-data-to-catalog/project-management/jira/) and [New Relic integration](https://docs.port.io/build-your-software-catalog/sync-data-to-catalog/apm-alerting/newrelic), the relevant blueprints will be automatically created in your portal.

If you chose not to install the integrations, you will need to create the blueprints manually:

<h3>Create the jira issue blueprint</h3>

1. Go to your [Builder](https://app.getport.io/settings/data-model) page.
2. Click on `+ Blueprint`.
3. Click on the `{...}` button in the top right corner, and choose "Edit JSON".
4. Add this JSON schema:
    
    <details>
    <summary><b>Jira Issue Blueprint (Click to expand)</b></summary>
    
    ```json
    {
      "identifier": "jiraIssue",
      "title": "Jira Issue",
      "icon": "Jira",
      "schema": {
        "properties": {
          "url": {
            "title": "Issue URL",
            "type": "string",
            "format": "url",
            "description": "URL to the issue in Jira"
          },
          "status": {
            "title": "Status",
            "type": "string",
            "description": "The status of the issue"
          },
          "issueType": {
            "title": "Type",
            "type": "string",
            "description": "The type of the issue"
          },
          "components": {
            "title": "Components",
            "type": "array",
            "description": "The components related to this issue"
          },
          "assignee": {
            "title": "Assignee",
            "type": "string",
            "format": "user",
            "description": "The user assigned to the issue"
          },
          "reporter": {
            "title": "Reporter",
            "type": "string",
            "description": "The user that reported to the issue",
            "format": "user"
          },
          "priority": {
            "title": "Priority",
            "type": "string",
            "description": "The priority of the issue"
          },
          "created": {
            "title": "Created At",
            "type": "string",
            "description": "The created datetime of the issue",
            "format": "date-time"
          },
          "updated": {
            "title": "Updated At",
            "type": "string",
            "description": "The updated datetime of the issue",
            "format": "date-time"
          }
        }
      },
      "calculationProperties": {},
      "relations": {}
    }
    
    ```
    
    </details>
    
5. Click "Save" to create the blueprint.

<h3>Create the New Relic Alert blueprint</h3>

1. Go to your [Builder](https://app.getport.io/settings/data-model) page.
2. Click on `+ Blueprint`.
3. Click on the `{...}` button in the top right corner, and choose "Edit JSON".
4. Add this JSON schema:
    
    <details>
    <summary><b>New Relic Alert Blueprint (Click to expand)</b></summary>
    
    ```json
    {
      "identifier": "newRelicAlert",
      "description": "This blueprint represents a New Relic alert in our software catalog",
      "title": "New Relic Alert",
      "icon": "NewRelic",
      "ownership": {
        "type": "Inherited",
        "title": "Owning Teams",
        "path": "alert_to_workload.service"
      },
      "schema": {
        "properties": {
          "priority": {
            "type": "string",
            "title": "Priority",
            "enum": [
              "CRITICAL",
              "HIGH",
              "MEDIUM",
              "LOW"
            ],
            "enumColors": {
              "CRITICAL": "red",
              "HIGH": "red",
              "MEDIUM": "yellow",
              "LOW": "green"
            }
          },
          "state": {
            "type": "string",
            "title": "State",
            "enum": [
              "ACTIVATED",
              "CLOSED",
              "CREATED"
            ],
            "enumColors": {
              "ACTIVATED": "yellow",
              "CLOSED": "green",
              "CREATED": "lightGray"
            }
          },
          "sources": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "title": "Sources"
          },
          "alertPolicyNames": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "title": "Alert Policy Names"
          },
          "conditionName": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "title": "Condition Name"
          },
          "link": {
            "type": "string",
            "title": "Link",
            "format": "url"
          },
          "description": {
            "type": "string",
            "title": "Description"
          },
          "activatedAt": {
            "type": "string",
            "title": "Activated at",
            "format": "date-time"
          }
        },
        "required": []
      },
      "mirrorProperties": {
        "cloud_resource_name": {
          "title": "Cloud Resource",
          "path": "cloud_resource.$title"
        },
        "port_service": {
          "title": "Service",
          "path": "alert_to_workload.service.$title"
        },
        "team": {
          "title": "Team",
          "path": "alert_to_workload.service.$team"
        },
        "workload_name": {
          "title": "Workload name",
          "path": "alert_to_workload.$title"
        },
        "pager_duty_service": {
          "title": "Pager duty service",
          "path": "alert_to_workload.service.pager_duty_service.$identifier"
        }
      },
      "calculationProperties": {},
      "aggregationProperties": {},
      "relations": {
        "alert_to_workload": {
          "title": "Workload",
          "target": "workload",
          "required": false,
          "many": false
        },
        "cloud_resource": {
          "title": "Cloud resource",
          "target": "newRelicCloudResource",
          "required": false,
          "many": false
        }
      }
    }
    
    ```
    
    </details>
    
5. Click "Save" to create the blueprint.

## Implementation

<Tabs>
  <TabItem value="webhook" label="Synced webhook" default>

    You can create Jira bugs by leveraging Port's **synced webhooks** and **secrets** to directly interact with the Jira's API. This method simplifies the setup by handling everything within Port.

    <h3>Add Port secrets</h3>

    <ExistingSecretsCallout integration="Jira" />

    To add these secrets to your portal:

    1. Click on the `...` button in the top right corner of your Port application.

    2. Click on **Credentials**.

    3. Click on the `Secrets` tab.

    4. Click on `+ Secret` and add the following secrets:
        - `JIRA_API_TOKEN` - Your Jira API token
        - `JIRA_USER_EMAIL` - The email of the Jira user that owns the API token
        - `JIRA_AUTH` - Base64 encoded string of your Jira credentials. Generate this by running:
          ```bash
          echo -n "your-email@domain.com:your-api-token" | base64
          ```
          Replace `your-email@domain.com` with your Jira email and `your-api-token` with your Jira API token.

          :::info One time generation
          The base64 encoded string only needs to be generated once and will work for all webhook calls until you change your API token.
          :::

    <h3>Set up self-service action</h3>

    Follow these steps to create the self-service action:

    1. Head to the [self-service](https://app.getport.io/self-serve) page.
    2. Click on the `+ New Action` button.
    3. Click on the `{...} Edit JSON` button.
    4. Copy and paste the following JSON configuration into the editor.

        <details>
        <summary><b>Report a bug from New Relic alert action (Webhook) (Click to expand)</b></summary>

        ```json
        {
          "identifier": "report_a_jira_bug_from_new_relic_alert",
          "title": "Report a bug from New Relic alert (Webhook)",
          "icon": "Jira",
          "description": "Report a Jira bug in Port based on a New Relic alert.",
          "trigger": {
            "type": "self-service",
            "operation": "DAY-2",
            "userInputs": {
              "properties": {
                "project": {
                  "title": "Project",
                  "description": "The Jira project where the bug will be created",
                  "icon": "Jira",
                  "type": "string",
                  "blueprint": "jiraProject",
                  "format": "entity"
                }
              },
              "required": [
                "project"
              ],
              "order": [
                "project"
              ]
            },
            "blueprintIdentifier": "newRelicAlert"
          },
          "invocationMethod": {
            "type": "WEBHOOK",
            "url": "https://<JIRA_ORGANIZATION_URL>.atlassian.net/rest/api/3/issue",
            "agent": false,
            "synchronized": true,
            "method": "POST",
            "headers": {
              "Authorization": "Basic {{.secrets.JIRA_AUTH}}",
              "Content-Type": "application/json"
            },
            "body": {
              "fields": {
                "project": {
                  "key": "{{.inputs.project.identifier}}"
                },
                "summary": "Bug | New Relic | {{.entity.title}}",
                "description": {
                  "version": 1,
                  "type": "doc",
                  "content": [
                    {
                      "type": "paragraph",
                      "content": [
                        {
                          "type": "text",
                          "text": "\n New Relic description: {{.entity.description}}\n\n New Relic link: {{.entity.link}}"
                        }
                      ]
                    },
                    {
                      "type": "paragraph",
                      "content": [
                        {
                          "type": "text",
                          "text": "Reported by: {{.trigger.by.user.email}}"
                        }
                      ]
                    }
                  ]
                },
                "issuetype": {
                  "name": "Bug"
                }
              }
            }
          },
          "requiredApproval": false
        }
        ```

        </details>

    5. Click `Save`.

    :::tip Configure your Jira url
    Replace `<JIRA_ORGANIZATION_URL>` in the webhook URL with your Jira organization URL (e.g., `example.atlassian.net`).
    :::

    Now you should see the `Report a bug from New Relic alert` action in the self-service page. 🎉

  </TabItem>

  <TabItem value="github" label="GitHub workflow">

      To implement this use-case using a GitHub workflow, follow these steps:

      <h3>Add GitHub secrets</h3>

      In your GitHub repository, [go to **Settings > Secrets**](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository) and add the following secrets:
      - `JIRA_API_TOKEN` - [Jira API token](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account) generated by the user.
      - `JIRA_BASE_URL` - The URL of your Jira organization. For example, https://your-organization.atlassian.net.
      - `JIRA_USER_EMAIL` - The email of the Jira user that owns the Jira API token.
      - `PORT_CLIENT_ID` - Your port `client id` [How to get the credentials](https://docs.port.io/build-your-software-catalog/sync-data-to-catalog/api/#find-your-port-credentials).
      - `PORT_CLIENT_SECRET` - Your port `client secret` [How to get the credentials](https://docs.port.io/build-your-software-catalog/sync-data-to-catalog/api/#find-your-port-credentials).

      <h3>Add GitHub workflow</h3>

      Create the file `.github/workflows/report-a-bug.yml` in the `.github/workflows` folder of your repository.

      <GithubDedicatedRepoHint/>

      <details>
      <summary><b>GitHub Workflow (Click to expand)</b></summary>

      ```yaml showLineNumbers
      name: Report a bug in jira

      on:
        workflow_dispatch:
          inputs:
            project:
              required: true
              type: string          
            description:
              required: true
              type: string
            short_title:
              required: true
              type: string
            port_context:
              required: true
              type: string

      jobs:
        create_jira_issue:
          runs-on: ubuntu-latest
          steps:
            - name: Login
              uses: atlassian/gajira-login@v3
              env:
                JIRA_BASE_URL: ${{ secrets.JIRA_BASE_URL }}
                JIRA_USER_EMAIL: ${{ secrets.JIRA_USER_EMAIL }}
                JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}

            - name: Inform searching of user in user list
              uses: port-labs/port-github-action@v1
              with:
                clientId: ${{ secrets.PORT_CLIENT_ID }}
                clientSecret: ${{ secrets.PORT_CLIENT_SECRET }}
                operation: PATCH_RUN
                runId: ${{ fromJson(inputs.port_context).run_id }}
                logMessage: |
                  Searching for user in organization user list... ⛴️

            - name: Search for reporter among user list
              id: search_for_reporter
              uses: fjogeleit/http-request-action@v1
              with:
                url: "${{ secrets.JIRA_BASE_URL }}/rest/api/3/user/search?query=${{ fromJson(inputs.port_context).triggered_by }}"
                method: "GET"
                username: ${{ secrets.JIRA_USER_EMAIL }}
                password: ${{ secrets.JIRA_API_TOKEN }}
                customHeaders: '{"Content-Type": "application/json"}'

            - name: Install jq
              run: sudo apt-get install jq
              if: steps.search_for_reporter.outcome == 'success'

            - name: Retrieve user list from search
              id: user_list_from_search
              if: steps.search_for_reporter.outcome == 'success'
              run: |
                selected_user_id=$(echo '${{ steps.search_for_reporter.outputs.response }}' | jq -r 'if length > 0 then .[0].accountId else "empty" end')
                selected_user_name=$(echo '${{ steps.search_for_reporter.outputs.response }}' | jq -r 'if length > 0 then .[0].displayName else "empty" end')
                echo "selected_user_id=${selected_user_id}" >> $GITHUB_OUTPUT
                echo "selected_user_name=${selected_user_name}" >> $GITHUB_OUTPUT

            - name: Inform user existence
              if: steps.user_list_from_search.outputs.selected_user_id != 'empty'
              uses: port-labs/port-github-action@v1
              with:
                clientId: ${{ secrets.PORT_CLIENT_ID }}
                clientSecret: ${{ secrets.PORT_CLIENT_SECRET }}
                operation: PATCH_RUN
                runId: ${{ fromJson(inputs.port_context).run_id }}
                logMessage: |
                  User found 🥹 Bug reporter will be ${{ steps.user_list_from_search.outputs.selected_user_name }}... ⛴️

            - name: Inform user inexistence
              if: steps.user_list_from_search.outputs.selected_user_id == 'empty'
              uses: port-labs/port-github-action@v1
              with:
                clientId: ${{ secrets.PORT_CLIENT_ID }}
                clientSecret: ${{ secrets.PORT_CLIENT_SECRET }}
                operation: PATCH_RUN
                runId: ${{ fromJson(inputs.port_context).run_id }}
                logMessage: |
                  User not found 😭 Bug will be created with default reporter ⛴️

            - name: Inform starting of jira issue creation
              uses: port-labs/port-github-action@v1
              with:
                clientId: ${{ secrets.PORT_CLIENT_ID }}
                clientSecret: ${{ secrets.PORT_CLIENT_SECRET }}
                operation: PATCH_RUN
                runId: ${{ fromJson(inputs.port_context).run_id }}
                logMessage: |
                  Creating a new Jira issue.. ⛴️

            - name: Create Jira issue
              id: create
              uses: atlassian/gajira-create@v3
              with:
                project: ${{ inputs.project }}
                issuetype: Bug
                summary: ${{inputs.short_title}}
                description: |
                  ${{inputs.description}}
                fields: |-
                  ${{ steps.user_list_from_search.outputs.selected_user_id != 'empty'
                    && format('{{"reporter": {{"id": "{0}" }}}}', steps.user_list_from_search.outputs.selected_user_id)
                    || '{}'
                  }}

            - name: Inform creation of Jira issue
              uses: port-labs/port-github-action@v1
              with:
                clientId: ${{ secrets.PORT_CLIENT_ID }}
                clientSecret: ${{ secrets.PORT_CLIENT_SECRET }}
                operation: PATCH_RUN
                link: ${{ secrets.JIRA_BASE_URL }}/browse/${{ steps.create.outputs.issue }}
                runId: ${{ fromJson(inputs.port_context).run_id }}
                logMessage: |
                  Jira issue with ID ${{ steps.create.outputs.issue }} created! ✅
      ```

      </details>

      <h3>Set up self-service action</h3>

      We will create a self-service action to handle creating bugs in Jira.
      To create a self-service action follow these steps:

      1. Head to the [self-service](https://app.getport.io/self-serve) page.
      2. Click on the `+ New Action` button.
      3. Click on the `{...} Edit JSON` button.
      4. Copy and paste the following JSON configuration into the editor.

          <details>
          <summary><b>Report a bug action (Click to expand)</b></summary>

          <GithubActionModificationHint/>

          ```json
          {
            "identifier": "jiraIssue_report_a_bug_from_newrelic",
            "title": "Report a bug from New Relic (GitHub actions)",
            "icon": "Jira",
            "description": "Report a bug in Port from a New Relic alert.",
            "trigger": {
              "type": "self-service",
              "operation": "DAY-2",
              "userInputs": {
                "properties": {
                  "project": {
                    "title": "Project",
                    "description": "The Jira project where the bug will be created",
                    "icon": "Jira",
                    "type": "string",
                    "blueprint": "jiraProject",
                    "format": "entity"
                  }
                },
                "required": [
                  "project"
                ],
                "order": [
                  "project"
                ]
              },
              "blueprintIdentifier": "newRelicAlert"
            },
            "invocationMethod": {
              "type": "GITHUB",
              "org": "<GITHUB_ORG>",
              "repo": "<GITHUB_REPO>",
              "workflow": "report-a-bug.yml",
              "workflowInputs": {
                "project": "{{.inputs.project.identifier}}",
                "description": "New Relic description: {{.entity.description}}\n\n New Relic link: {{.entity.link}}\n\n Reported by: {{.trigger.by.user.email}}",
                "short_title": "Bug | New Relic | {{.entity.title}}",
                "port_context": {
                  "run_id": "{{ .run.id }}",
                  "triggered_by": "{{ .trigger.by.user.email }}"
                }
              },
              "reportWorkflowStatus": true
            },
            "requiredApproval": false
          }
          ```

          </details>

      5. Click `Save`.

      Now you should see the `Report a bug from New Relic (gitHub actions)` action in the self-service page. 🎉

  </TabItem>
</Tabs>

## Let's test it!

1. Head to the [self-service page](https://app.getport.io/self-serve) of your portal

2. Choose either the GitHub workflow or webhook implementation:
   - For webhook: Click on `Report a bug from New Relic alert (Webhook)`
   - For GitHub workflow: Click on `Report a bug from New Relic alert (GitHub actions)`

3. Fill in the bug details:
   - Select the New Relic alert that will trigger the creation of the Jira bug
   - Select the Jira project where the bug will be created

4. Click on `Execute`

5. Done! wait for the bug to be created in Jira
