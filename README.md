# check_azure_uic
> `check_azure_uic` is a Replacement Plugin for Opsview and Nagios check_azure. Provides an lower level and easily extensible plugin
>for monitoring Azure

> Currently `check_azure_uic` only provides replacement checks for `Az.Container.Serv.Kube.*`

## Features
   ✔ Check Amount of Available Memory in AKS
   
   ✔ Check Amount of Available CPU Cores for AKS
   
   ✔ Check Number of Ready Pods for AKS
   
   ✔ Check the Condition of an AKS Cluster
   
   ✔ Easily Extensible
   
   ✔ Secure, fast connection
   
## Install - Nagios

- Install python3 and python3-pip for your distribution
- `pip3 install -r requirements.txt`
- Copy check_azure_uic to libexec / plugins folder
- Add Command to your config

`commands.cfg`
```
define command {
    command_name    check_https
    command_line    $USER1$/check_azure_uic -m $ARG1$ --tenant-id $ARG2$ --client-id $ARG3$ --secret-key $ARG4$ --subscription-id $ARG5$ --resource-group $ARG6$ --resource-name $ARG7$ -w $ARG8$ -c $ARG9$
}
```
- Add Command to a Host with IP 127.0.0.1 or Localhost

## Install - Opsview

- Install python3 and python3-pip for your distribution
- `pip3 install -r requirements.txt`
- Install Azure Opspack (This is installed by default on most modern installations)
- Import into Opsview under `Configuration -> Monitoring Plugins`
- Add the service checks you wish to use `Configuration -> Service Chacks`
- Change plugin arguments 
```
--mode <Plugin Mode> --subscription-id '%AZURE_CREDENTIALS:1%' --client-id '%AZURE_CREDENTIALS:2%' --secret-key '%AZURE_CREDENTIALS:3%' --tenant-id '%AZURE_CREDENTIALS:4%' --resource-group '%AZURE_RESOURCE_DETAILS:1%' --resource-name '%AZURE_RESOURCE_DETAILS:2%'
```
- In `Configuration -> Variables` add credentials to `Azure Credentials`
- (Optional) Add Service Checks to Host Template
- Create a host group
- Create a host with IP as `127.0.0.1` or the FQDN of the server
- Configure details of the host to your preferences
- Under `Variables` add `AZURE_RESOURCE_DETAILS`
- Check `Override Resource Group` and `Override Resource Name` and input your configuration information
- Submit and Apply Changes
- Monitor!

## Usage
```
Usage: check_azure_uic -m MODE [-h] (Optional Arguments)
Monitors Microsoft Azure UIC:

Valid Modes:
Az.Container.Serv.Kube.CPU
Az.Container.Serv.Kube.Condition
Az.Container.Serv.Kube.Memory
Az.Container.Serv.Kube.Pods.Ready

Options:
  -h, --help            show this help message and exit
  -c CRITICAL_LIMIT, --critical=CRITICAL_LIMIT
  -w WARNING_LIMIT, --warning=WARNING_LIMIT
  -m MODE, --mode=MODE  Required Argument - Defines Mode
  --tenant-id=TENANT_ID
                        Azure Tenant ID
  --client-id=CLIENT_ID
                        Azure Client ID
  --secret-key=SECRET_KEY
                        Azure Secret Key
  --subscription-id=SUBSCRIPTION_ID
                        Azure Subscription ID
  --resource-group=RESOURCE_GROUP
                        Azure Resource Group ID
  --resource-name=RESOURCE_NAME
                        Azure Resource Name

```

Examples:
```
check_azure_uic -m Az.Container.Serv.Kube.CPU --tenant-id <TenantId> --client-id <ClientId> --secret-key <SecretKey> --subscription-id <SubId> --resource-group <RG Name> --resource-name <ResourceName>
check_azure_uic -m Az.Container.Serv.Kube.Condition --tenant-id <TenantId> --client-id <ClientId> --secret-key <SecretKey> --subscription-id <SubId> --resource-group <RG Name> --resource-name <ResourceName> -w 50 -c 25
```

Output:
```
OK - Cluster Health: CPU Cores Available - 30.0
OK - Cluster Health: Containers up - 100.0%
```

## Custom Template

> Test API Requests using Postman or your preferred request generator [2 Min Setup Tutorial](https://blog.jongallant.com/2017/11/azure-rest-apis-postman/)
>
> Any variables denoted by <> needs to be changed

Location: `# Define Azure Actions`
```python
def <Mode Function Name>(auth_token, subscriptionId, <Other Args>..., warning_limit = <default-val>, critical_limit = <default-val>):
    api_version = <API Version>
    # Set auth header
    azure_header = {
        "Content-Type": "application/json",
        "Accept": "application/json",
        "Authorization": f"Bearer {auth_token}"
    }

    # Set parameters for Azure
    metricParams = f"?api-version={api_version}"

    # Ensure URL is correct and accessible
    try:
        r = requests.get(
            # Modify <API Path to the API URL>
            f"https://management.azure.com/subscriptions/{subscriptionId}/<API Path>{metricParams}",
            headers=azure_header)
    except ConnectionError as e:
        nagios_out("Could not connect - " + str(e), 3)
    except TimeoutError:
        nagios_out("Connection timed out", 3)

    if r.status_code != 200:
        nagios_out("Unsuccessful request, status: " + str(r.status_code) + ", error: " + str(r.json()), 3)

    # If this is throwing errors, then most likely your time range is invalid
    # Change JSON Path to the path to the variable
    try:
        return_data = r.json()
        <API Data> = return_data<JSON Path>
    except ValueError:
        nagios_out("Error parsing JSON return - " + return_data, 2)
    except IndexError:
        nagios_out("Error parsing JSON return - " + return_data, 2)
    
    # Change <API Data> to a useful variable name
    # Change strings to describe the data
    if <API Data> <= critical_limit:
        nagios_out(f"Cluster Health: CPU Cores Available - {<API Data>} - Critical Limit Reached", 2)
    elif <API Data> <= warning_limit:
        nagios_out(f"Cluster Health: CPU Cores Available - {<API Data>} - Warning Limit Reached", 1)

    nagios_out(f"Cluster Health: CPU Cores Available - {<API Data>}", 0)
```

Location: `# Add new options here`

```python
parser.add_option("Short Name", "<Long Name>", dest="<Name to use for parser>", help="<Add Help Item Information>")
```

Location: `# Parse Arguments Here`

```python
<new var name> = options.<Name to use for parser>
```

Location: `# Add new modes here`

```python
elif mode == "<Mode Name>":
    # Check Required Arguments
    if <required arg> is None or ...:
        nagios_out("Mode requires resource-name, resource-group", 3)
    elif warning_limit is not None and critical_limit is not None:
        <Mode Function Name>(api_key, subscriptionId, <Extra Args>, warning_limit, critical_limit)
    else:
        <Mode Function Name>(api_key, subscriptionId, <Extra Args>)
```

## Troubleshooting

`I am getting a python error`

+ At the top of the `check_azure_uic` change the python path to your server's python path

`My Host is showing as down`

+ Change the hostname to the FQDN or localhost

`I Keep getting random API errors`

+ Check your credentials and permissions

`I have an issue not listed here!`

+ Open an issue, I will attempt to answer as soon as possible!

## Licensing and Authors

### Authors

- Christian Bingman

#### NOTICE

Copyright (c) University of Illinois at Chicago. All rights reserved.