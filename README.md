### slogen

CLI tool to generate SLO dashboards, monitors & scheduled views from OpenSLO configs. Currently only supports sumo as
data source and target.

For a given config it will create the following content via sumo terraform provider

- Scheduled view to generate the aggregated SLI data
- Dashboard to track availability, burn rate and budget remaining
- Multi-Window, Multi-BurnRate monitors
- Global dashboard, to track availability across all services in a single view
- The content are grouped into folders of the service they belong to

#### sample config

```yaml
apiVersion: openslo/v1alpha
kind: SLO
metadata:
  displayName: SLO descriptive name
  name: slo-minimal-name   # only '-' is allowed apart from alphanumeric chars, '-' not allowed in start or end
spec:
  service: my-service
  description: service description to be added in dashboard text panel
  budgetingMethod: Occurrences
  objectives:
    - ratioMetrics:
        total: # sumo query to filter out all the messages counting to valid request
          source: sumologic
          queryType: Logs
          query: '_sourceCategory=my-service | where api_path="/login"'
        good: # condition to filter out healthy request/events
          source: sumologic
          queryType: Logs
          query: '(responseTime) < 500 and (statusCode matches /[2-3][0-9]{2}/ )'
        incremental: true
      displayName: delay less than 350
      target: 0.98
  timeWindows:
    - count: 1
      isRolling: true
      unit: Day
fields: # fields from log to retain
  region: "aws_region"    # log field as it is
  deployment: 'if(isNull(deployment),"dev",deployment)' # using an expression
labels:
  tier: 0                 # static labels to include in SLI view, that are not present in the log messages
burnRateAlerts: # Multiwindow, Multi-Burn-Rate Alerts, explained here https://sre.google/workbook/alerting-on-slos/ 
  - shortWindow: '10m' # the smaller window
    shortLimit: 14  # limit for the burn rate ratio, 14 denotes the error consumed in the window were 14 times the allowed number  
    longWindow: '1h'
    longLimit: 14
    notifications: # one or more notification channels
      - connetionType: 'Email'
        recipients: 'youremailid@email.com'
        triggerFor:
          - Warning
          - ResolvedWarning
      - connetionType: 'PagerDuty'
        connectionID: '1234abcd'  # id of pagerduty connection created in sumo
        triggerFor:
          - Critical
          - ResolvedCritical



```

#### Getting the tool

##### install with go1.17 as `go install github.com/agaurav/slogen@latest`

latest golang release can be installed by using the direction here : https://github.com/udhos/update-golang#usage

##### Get the latest binary from [release page](https://github.com/agaurav/slogen/releases)

#### Using the tool

Set the sumologic auth : https://registry.terraform.io/providers/SumoLogic/sumologic/latest/docs#environment-variables

```
Usage:
slogen [paths to yaml config]... [flags]
slogen [command]

Examples:
slogen service/search.yaml 
slogen ~/team-a/slo/ ~/team-b/slo ~/core/slo/login.yaml 
slogen ~/team-a/slo/ -o team-a/tf
slogen ~/team-a/slo/ -o team-a/tf --apply 

Available Commands:
completion generate the autocompletion script for the specified shell docs A brief description of your command help Help
about any command new create a sample config from given profile validate A brief description of your command

Flags:
-o, --out string        :   output directory where to create the terraform files (default "tf")
-d, --dashboardFolder   :   string output directory where to create the terraform files (default "slogen-tf-dashboards")
-m, --monitorFolder     :   string output directory where to create the terraform files (default "slogen-tf-monitors")
-i, --ignoreErrors      :   whether to continue validation even after encountering errors 
-p, --plan              :   show plan output after generating the terraform config 
-a, --apply             :   apply the generated terraform config as well 
-c, --clean             :   clean the old tf files for which openslo config were not found in the path args 
-h, --help              :   help for slogen


Use "slogen [command] --help" for more information about a command. Example config with inline comment explaining the
various fields

```

#### Where are the content created

- Dashboards by default are created in a folder `slogen-tf-dashboards`. A different folder can be set with the
  flag `-d or --dashboardFolder` followed by name of the folder. It will take some time for historical data to be
  calculated for new views for dashboards to be accurate.
- Monitors by default are created in a folder `slogen-tf-monitors`. A different folder can be set with the
  flag `-m or --monitorFolder` followed by name of the folder.
- Scheduled view are created with a prefix `slogen_tf_`, this can't be changed as of now.
- `SLO Overview` dashboard is created at the root of Dashboard folder specified.

#### Limitations

- only support Sumologic Logs as data source.
- Only `Occurrences` based `budgetingMethod` is handled. support for `Timeslices` is work in progress.
- Alerting on SLO, burn-rate can be configured only up-to `24h`. Tracking them via dashboard is still possible for up-to
  31 days.


