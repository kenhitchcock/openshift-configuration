Create Machine sets
=========

  * Currently only available for AWS and AZURE.

This role will create the INFRA-node machine sets for your cluster including the MCP.

Requirements
------------

You need to set the correct variables as show below for cloud provider, region and OCP version.

Role Variables
--------------

Update the global variable file in the root of this code ````group_vars/all
make sure you set the correct cloud provider to "true"

````
aws: "false"
azure: "true"
vmware: "false"
````

To use this role, you need to set some variables under ````vars/main.yaml````.

Now add the region spectific information for where you want to build your INFRA nodes. When the playbook runs, it will loop over these values so add more regions as required.
````
# Machine set variables per cloud provider.
azure_machinesets:
  - name: infra
    az: 1
  - name: infra
    az: 2
  - name: infra
    az: 3

aws_machinesets:
  - name: infra
    az: a
  - name: infra
    az: b
  - name: infra
    az: c
````

Dependencies
------------

The **local-facts** role must run before this role and populated the region information in ````/etc/ansible/facts.d/local.fact````

Example Playbook
----------------

This role is from the ```deploy_roles.yaml```` file. Just un=hash the machineset section.

Example run
-----------

Expand for an example run of the playbook:

<details>
 <summary>Grafana setup</summary>
  <p>

````
$ ansible-playbook deploy_roles.yaml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Setup local Facts] ************************************************************************************************************************************
[WARNING]: While constructing a mapping from /home/centos/openshift-day2/group_vars/all, line 3, column 1, found a duplicate dict key (kubeconfig). Using
last defined value only.

TASK [Gathering Facts] **************************************************************************************************************************************
ok: [localhost]

TASK [local-facts : Create custom fact directory] ***********************************************************************************************************
ok: [localhost]

TASK [local-facts : Insert custom fact file] ****************************************************************************************************************
changed: [localhost]

TASK [local-facts : Insert custom fact script] **************************************************************************************************************
ok: [localhost]

TASK [local-facts : add Instance facts] *********************************************************************************************************************
changed: [localhost]

TASK [local-facts : local facts] ****************************************************************************************************************************
ok: [localhost] => {
    "ansible_local": {
        "local": {
            "localfacts": {
                "ami_id": "",
                "clusternumber": "qgb2h",
                "full_cluster_name": "cluster-1234-qgb2h",
                "kubeconfig": "/home/centos/cluster-build/auth/kubeconfig",
                "region": "uksouth",
                "region_av_zone": "",
                "region_zone": ""
            }
        }
    }
}

TASK [local-facts : reload facts] ***************************************************************************************************************************
ok: [localhost]

PLAY [run OCP roles] ****************************************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************************
ok: [localhost]

TASK [machinesets : Create cluster build DIR] ***************************************************************************************************************
ok: [localhost]

TASK [machinesets : Create AZURE build config] **************************************************************************************************************
changed: [localhost] => (item={u'zoneaz': 1, u'name': u'infra', u'zone': 1})
changed: [localhost] => (item={u'zoneaz': 2, u'name': u'infra', u'zone': 2})

TASK [machinesets : Create AZURE machineset] ****************************************************************************************************************
changed: [localhost] => (item={u'zoneaz': 1, u'name': u'infra', u'zone': 1})
changed: [localhost] => (item={u'zoneaz': 2, u'name': u'infra', u'zone': 2})

TASK [machinesets : Create AWS build config] ****************************************************************************************************************
skipping: [localhost] => (item={u'zoneaz': u'2a', u'az': u'a', u'name': u'infra', u'zone': 2})
skipping: [localhost] => (item={u'zoneaz': u'2b', u'az': u'b', u'name': u'infra', u'zone': 2})

TASK [machinesets : Create a Deployment by reading the definition from a local file] ************************************************************************
skipping: [localhost] => (item={u'zoneaz': u'2a', u'az': u'a', u'name': u'infra', u'zone': 2})
skipping: [localhost] => (item={u'zoneaz': u'2b', u'az': u'b', u'name': u'infra', u'zone': 2})

PLAY RECAP **************************************************************************************************************************************************
localhost                  : ok=11   changed=4    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
````

</p></details>

OC command showing the machine/sets being created:

````
$ oc get machineset -n openshift-machine-api
NAME                                 DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster-1234-qgb2h-infra-uksouth1    1         1                             56s
cluster-1234-qgb2h-infra-uksouth2    1         1                             55s
cluster-1234-qgb2h-worker-uksouth1   1         1         1       1           85m
cluster-1234-qgb2h-worker-uksouth2   1         1         1       1           85m
cluster-1234-qgb2h-worker-uksouth3   1         1         1       1           85m

$ oc get machine -n openshift-machine-api
NAME                                       PHASE          TYPE              REGION    ZONE   AGE
cluster-1234-qgb2h-infra-uksouth1-48ztv    Provisioning   Standard_D2s_v3   uksouth   1      60s
cluster-1234-qgb2h-infra-uksouth2-vs5vq    Provisioning   Standard_D2s_v3   uksouth   2      59s
cluster-1234-qgb2h-master-0                Running        Standard_D4s_v3   uksouth   3      85m
cluster-1234-qgb2h-master-1                Running        Standard_D4s_v3   uksouth   2      85m
cluster-1234-qgb2h-master-2                Running        Standard_D4s_v3   uksouth   1      85m
cluster-1234-qgb2h-worker-uksouth1-f2ft6   Running        Standard_D2s_v3   uksouth   1      80m
cluster-1234-qgb2h-worker-uksouth2-c8lc6   Running        Standard_D2s_v3   uksouth   2      80m
cluster-1234-qgb2h-worker-uksouth3-xss92   Running        Standard_D2s_v3   uksouth   3      80m

$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT
infra    rendered-infra-19532f16af8effebc7dfd13d42cbe759    True      False      False      2              2                   2                     0
master   rendered-master-2cef252b75b9a893508a4bb62cfbf310   True      False      False      3              3                   3                     0
worker   rendered-worker-19532f16af8effebc7dfd13d42cbe759   True      False      False      3              3                   3                     0

````


License
-------

BSD

Author Information
------------------


