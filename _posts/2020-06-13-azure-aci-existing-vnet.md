---
layout: post
title:  "Azure Container Instances into Existing Vnets"
date:   2020-06-13 12:00:32 -0500
categories: azure
tags: azure container devops
---
Quick one that I found kind of awkwardly documented by Azure, how to spin up a container in an existing vnet. The documentation from Microsoft on this is found at: <https://docs.microsoft.com/en-us/azure/container-instances/container-instances-vnet>

They don't really go into it much at an example level, it just says 'give it the name of the vnet and subnet', the response I kept getting created **another** vnet with the same name, inside the Resource Group. 


## The Scenario
So right now, my proposed mini setup is as follows:

Resource Group: "ALASH-USC-VNET"
- Virtual Network: "ALASH-USC-VNET" - 10.0.0.0/16
- Subnet: "ACI" - 10.0.0.0/24 (Subnet has no association to Container.. Yet)

Resource Group: "ALASH-USC-ACITEST"
- Azure Container Instance: "Hello World" running inside the ACI subnet above.


The outcome is expected that the 'ACI' subnet turns into a container-associated subnet, and the commands provision a network profile, and allow this subnet to become a dedicated azure container instance subnet.

&nbsp;
&nbsp;
&nbsp;
&nbsp;

## The Commands
### Command 1, 'Just give it the vnet/subnet names...':
```
az container create \
  --resource-group ALASH-USC-ACITEST \
  --name aci-nginx \
  --image nginx:latest \
  --restart-policy never \
  --vnet "ALASH-USC-ACI" \
  --subnet "ACI"
  ```

![useful image]({{ site.url }}/assets/azure-aci-01.png)
*two vnets, same name.. hmmm...*


The above command has ended up creating a new vnet inside the ACITEST resource group, disregarding the fact that one already exists. While this kind of design is relatively correct in most modern hub + spoke vnet setups (where the app would have its own vnet/subnets), The current design this project has gone with is more dedicated 'larger' vnets for entire tenants... So how do we get these containers into the 'tenant hub' vnet.

&nbsp;
&nbsp;
&nbsp;
&nbsp;


### Command 2: 'Okay, lets get more specific.'
```
az container create \
  --resource-group ALASH-USC-ACITEST \
  --name aci-nginx \
  --image nginx:latest \
  --restart-policy never \
  --vnet "/subscriptions/1231231312312312312312312/resourceGroups/ALASH-USC-VNET/providers/Microsoft.Network/virtualNetworks/ALASH-USC-ACI" \
  --subnet "/subscriptions/1231231312312312312312312/resourceGroups/ALASH-USC-VNET/providers/Microsoft.Network/virtualNetworks/ALASH-USC-ACI/subnets/ACI"
```

denote here the *--vnet* and *--subnet* have been scoped to the Resource ID


This has given us the expected result:
![useful image]({{ site.url }}/assets/azure-aci-02.png)

So, it seems it REALLY wants the vnet/subnet ids in their full. I guess it makes sense, the `az create` must be just looking inside its own Resource Group, and to get it to traverse out we gotta give it proper IDs. As a side-note, I do wish that Microsoft did make these ID's a little more succinct (AWS ARN, looking at you...).

&nbsp;
&nbsp;
&nbsp;
&nbsp;

### Command 3, can we do this with YAML?

The next thought is if we can define such things with the YAML 'k8esque' declaration that ACI supports?


The answer (as from what I can tell with the doco anyway) seems to be no. As per the YAML reference: <https://docs.microsoft.com/en-us/azure/container-instances/container-instances-reference-yaml> I note there is a reference to *networkProfile* meaning I could give an existing Network Profile ID (like the one that gets created above with `az create`) and it will use that as a base? Lets give that a shot:

#### Step 1: Get our Network Profile
So, our profile from the above we can retrieve with: `az network profile list` and scope the query down to only the IDs

```
Adams-MacBook-Pro:terraform adamlash$ az network profile list --query [0].id --output tsv
/subscriptions/1231231312312312312312312/resourceGroups/ALASH-USC-VNET/providers/Microsoft.Network/networkProfiles/aci-network-profile-ALASH-USC-ACI-ACI
```

#### Step 2: Create our YAML

We can use this in our YAML:


**aci-nginx-yaml.yaml**
```
apiVersion: '2018-10-01'
identity: null
location: centralus
name: aci-nginx-yaml
properties:
  containers:
  - name: aci-nginx-yaml
    properties:
      environmentVariables: []
      image: nginx:latest
      ports:
      - port: 80
        protocol: TCP
      resources:
        requests:
          cpu: 1.0
          memoryInGB: 1.5
  ipAddress:
    ports:
    - port: 80
      protocol: TCP
    type: Private
  networkProfile:
    id: /subscriptions/1231231312312312312312312/resourceGroups/ALASH-USC-VNET/providers/Microsoft.Network/networkProfiles/aci-network-profile-ALASH-USC-ACI-ACI
  osType: Linux
  restartPolicy: Never
tags: {}
```

Denote here the networkProfile, specifying these network settings.


*as a bonus, you can also export these yaml files from existing deployments with the az cli via `az container export -g "ResourceGroup"  --name "ContainerName" -f output.yaml`*

#### Step 3: Deploy
And deploy with `az container create --resource-group ALASH-USC-ACITEST --file aci-nginx-yaml.yaml`

&nbsp;
&nbsp;



![useful image]({{ site.url }}/assets/azure-aci-03.png)


&nbsp;
&nbsp;
&nbsp;
&nbsp;


## The Conclusion
So, it seems that this pattern is possible but you need to do some housekeeping
- Specify the IDs of the Subnet/Vnet you want to deploy into
- If you want to use YAML, you need an exisiting Network Profile to point at
- You can create network profiles by using the `az container create` command with all the relevant flags to create your container


It would be pretty great if Azure let us create Network Profiles independently, there must be a reason why it's a little akward. I may be missing something obvious though, who knows! Maybe there will be an update to this little post proving how wrong I was.







