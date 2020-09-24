#**Add Additional Worker**

In this section we're going to demonstrate how to expand a Baremetal IPI environment with an additional 3rd **worker** node, i.e. a node that will just run workloads, not cluster services. The eagle eyed amongst you may notice that we listed _"NUM\_WORKERS=2"_ in our initial configuration file, but we've actually got a spare unused "baremetal" host in the environment you're working in. We simply need to tell the baremetal operator about it, and get it to deploy RHCOS and OpenShift onto it.

We need to supply a baremetal node defintion to the baremetal operator to do this, and thankfully in this lab there is a handy way to achieve this.  First we need to obtain the ipmi port of the **worker-2** node by using the following one liner against the install-config.yaml we used to deploy the initial cluster:

~~~bash
[cloud-user@provision scripts]$ for PORT in 6200 6201 6202 6203 6204 6205; do grep -q $PORT ~/scripts/install-config.yaml; if [ $? -eq 1 ]; then echo $PORT; fi; done
6203
~~~

One we have the port number lets create the following baremetal node definition file.  Make sure to update the port number to in the ipmi address line to the value you obtained above.  All other values, including the MAC address can remain the same since this lab is running in a virtualized environment.

~~~bash
[cloud-user@provision scripts]$ cat << EOF > ~/bmh.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: worker-2-bmc-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cmVkaGF0
---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: worker-2
spec:
  online: true
  bootMACAddress: de:ad:be:ef:00:52
  hardwareProfile: openstack
  bmc:
    address: ipmi://10.20.0.3:6203
    credentialsName: worker-2-bmc-secret
EOF
~~~

Let's look at the file that it created:

~~~bash
$ cat ~/bmh.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: worker-2-bmc-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cmVkaGF0
---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: worker-2
spec:
  online: true
  bootMACAddress: de:ad:be:ef:00:52
  hardwareProfile: openstack
  bmc:
    address: ipmi://10.20.0.3:6203
    credentialsName: worker-2-bmc-secret
~~~

You'll see that this is set to create two different resources, one is a `secret` that contains a base64 encoded username/password for IPMI access for the baremetal host, and then a `BareMetalHost` object (linked to the secret) that will define the node within the cluster itself, which is enough for the baremetal operator to kick off the deployment.

Let's now create these resources:

~~~bash

[cloud-user@provision ~]$ oc create -f ~/bmh.yaml -n openshift-machine-api

~~~

Immediately you should this being reflected in as a new `BareMetalHost`, note the new entry with a status of "**inspecting**":

~~~bash
[cloud-user@provision ~]$ oc get baremetalhosts -n openshift-machine-api
NAME       STATUS   PROVISIONING STATUS      CONSUMER                     BMC                     HARDWARE PROFILE   ONLINE   ERROR
master-0   OK       externally provisioned   schmaustech-master-0         ipmi://10.20.0.3:6202                      true     
master-1   OK       externally provisioned   schmaustech-master-1         ipmi://10.20.0.3:6201                      true     
master-2   OK       externally provisioned   schmaustech-master-2         ipmi://10.20.0.3:6205                      true     
worker-0   OK       provisioned              schmaustech-worker-0-2czx7   ipmi://10.20.0.3:6204   openstack          true     
worker-1   OK       provisioned              schmaustech-worker-0-qgmhk   ipmi://10.20.0.3:6200   openstack          true     
worker-2   OK       inspecting                                            ipmi://10.20.0.3:6203                      true  
~~~

You'll also notice this is listed (and managed) by Ironic that's running on the cluster as part of the baremetal operator (note the "inspect wait" status as per the above):

~~~bash
[cloud-user@provision ~]$ export OS_URL=http://172.22.0.3:6385; export OS_TOKEN=fake-token
[cloud-user@provision ~]$ openstack baremetal node list
+--------------------------------------+----------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name     | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+----------+--------------------------------------+-------------+--------------------+-------------+
| 2d68e1cd-8d85-4dcc-a289-53c2770f3abb | master-1 | f2398c03-60f4-44be-84cb-d03d22509345 | power on    | active             | False       |
| 41ce9485-ae8b-42cc-ad85-4771e41e0af1 | master-0 | a30ce010-4c29-4fe7-9eb1-2b4f70f9b618 | power on    | active             | False       |
| ca9ca2b2-8de9-4521-8b94-514222400631 | master-2 | f16b8768-6f3f-44b1-8135-7b159984e44d | power on    | active             | False       |
| cffbddd5-2a1c-430f-8ffe-a5409a0f3538 | worker-0 | 099f8902-117e-4d58-8e67-7b8e700582e6 | power on    | active             | False       |
| 7eaa2bdb-0102-419c-9f5b-0f7e35693d6b | worker-1 | 656cb417-61b0-4a02-b4f5-393726536002 | power on    | active             | False       |
| e8d73d35-9379-4c73-87f3-2454a1e2c575 | worker-2 | None                                 | power on    | inspect wait       | False       |
+--------------------------------------+----------+--------------------------------------+-------------+--------------------+-------------+
~~~

This additional worker node will now be inspected, data will be gathered about the system such as number of nic's, storage disks, CPU, memory, and so on. The system will then sit in a holding pattern - all we've done is add it as a baremetal host, NOT an OpenShift worker, nor has it actually had any components installed on it, it's merely just running from a basic ramdisk for now.

What I recommend is that you watch the list of `BareMetalHosts` every 10s or so to see when it has finished this inspection step (you're looking for the node state of the worker to be '**ready**'):

~~~bash
[cloud-user@provision ~]$ watch -n10 oc get baremetalhosts -n openshift-machine-api
(...)

(Ctrl-C to stop)
~~~

Now you should see the machine as '**ready**' in the list (you likely saw it in the 'watch' as above, but it's worth confirming):

~~~bash
[cloud-user@provision ~]$ oc get baremetalhosts/worker-2 -n openshift-machine-api
NAME       STATUS   PROVISIONING STATUS   CONSUMER   BMC                     HARDWARE PROFILE   ONLINE   ERROR
worker-2   OK       ready                            ipmi://10.20.0.3:6203   openstack          true  
~~~

For it to become managed by OpenShift as a `Machine` and for it to become a `Node`, so we can run applications on it, we need to scale the respective `MachineSet` up; this will kick off the process for deploying CoreOS, installing the OpenShift components, configuring the components to talk to our cluster, and ensuring everything is running properly. When the baremetal opertator is deployed, it creates a `MachineSet` for worker nodes automatically for us:

~~~bash
[cloud-user@provision ~]$ oc -n openshift-machine-api get machineset
NAME                   DESIRED   CURRENT   READY   AVAILABLE   AGE
schmaustech-worker-0   2         2         2       2           77m
~~~

Much like scaling other platforms that are integrated with OpenShift 4.x (e.g. AWS), to dynamically scale our cluster from available resources we simply need to increase the number of 'replicas' to our `MachineSet`. The primary difference between scaling a baremetal cluster and a cluster that utilises the public cloud is that you have to define all of your nodes in baremetal whilst with the public cloud it can spawn new VM's on-demand through the public cloud API.

Let's now scale our cluster:

~~~bash
[cloud-user@provision ~]$ oc -n openshift-machine-api scale machineset schmaustech-worker-0 --replicas=3
machineset.machine.openshift.io/schmaustech-worker-0 scaled
~~~

Now if you check the `BareMetalHosts` you should see a slightly different state for our worker:

~~~bash
[cloud-user@provision ~]$ oc get baremetalhosts -n openshift-machine-api
NAME       STATUS   PROVISIONING STATUS      CONSUMER                     BMC                     HARDWARE PROFILE   ONLINE   ERROR
master-0   OK       externally provisioned   schmaustech-master-0         ipmi://10.20.0.3:6202                      true     
master-1   OK       externally provisioned   schmaustech-master-1         ipmi://10.20.0.3:6201                      true     
master-2   OK       externally provisioned   schmaustech-master-2         ipmi://10.20.0.3:6205                      true     
worker-0   OK       provisioned              schmaustech-worker-0-2czx7   ipmi://10.20.0.3:6204   openstack          true     
worker-1   OK       provisioned              schmaustech-worker-0-qgmhk   ipmi://10.20.0.3:6200   openstack          true     
worker-2   OK       provisioning             schmaustech-worker-0-kbwlb   ipmi://10.20.0.3:6203   openstack          true   
~~~

You should also notice that Ironic gives us an updated output to say that it's actually **deploying**, and it's at this point that we deploy CoreOS on our node:

~~~bash
[cloud-user@provision ~]$ openstack baremetal node list
+--------------------------------------+----------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name     | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+----------+--------------------------------------+-------------+--------------------+-------------+
| 2d68e1cd-8d85-4dcc-a289-53c2770f3abb | master-1 | f2398c03-60f4-44be-84cb-d03d22509345 | power on    | active             | False       |
| 41ce9485-ae8b-42cc-ad85-4771e41e0af1 | master-0 | a30ce010-4c29-4fe7-9eb1-2b4f70f9b618 | power on    | active             | False       |
| ca9ca2b2-8de9-4521-8b94-514222400631 | master-2 | f16b8768-6f3f-44b1-8135-7b159984e44d | power on    | active             | False       |
| cffbddd5-2a1c-430f-8ffe-a5409a0f3538 | worker-0 | 099f8902-117e-4d58-8e67-7b8e700582e6 | power on    | active             | False       |
| 7eaa2bdb-0102-419c-9f5b-0f7e35693d6b | worker-1 | 656cb417-61b0-4a02-b4f5-393726536002 | power on    | active             | False       |
| e8d73d35-9379-4c73-87f3-2454a1e2c575 | worker-2 | 4bff846f-fdbc-4c7e-8ab4-75865e327a77 | power on    | deploying          | False       |
+--------------------------------------+----------+--------------------------------------+-------------+--------------------+-------------+
~~~

If you re-run an extended `watch` command you can keep track of the process across multiple components:

~~~bash
[cloud-user@provision ~]$ watch -n10 "openstack baremetal node list && echo \
	&& oc get baremetalhosts -n openshift-machine-api && echo \
	&& oc get nodes"

(...)

(Ctrl-C to stop)
~~~

> **NOTE**: This process should take around 10 minutes before the new "baremetal" machine shows up as a new node in the list. You may also want to enlarge your terminal window so everything displays clearly.

Now let's verify the node status:

~~~bash
[cloud-user@provision ~]$ oc get nodes
NAME                                            STATUS     ROLES    AGE   VERSION
master-0.schmaustech.students.osp.opentlc.com   Ready      master   84m   v1.18.3+2cf11e2
master-1.schmaustech.students.osp.opentlc.com   Ready      master   86m   v1.18.3+2cf11e2
master-2.schmaustech.students.osp.opentlc.com   Ready      master   84m   v1.18.3+2cf11e2
worker-0.schmaustech.students.osp.opentlc.com   Ready      worker   63m   v1.18.3+2cf11e2
worker-1.schmaustech.students.osp.opentlc.com   Ready      worker   62m   v1.18.3+2cf11e2
worker-2.schmaustech.students.osp.opentlc.com   NotReady   worker   26s   v1.18.3+2cf11e2
~~~

It may demonstrate a '**NotReady**' status, if it does it will likely be because all of the necessary configuration will not have yet been applied, nor will it have all of the pods/containers running yet. Give it some time and all node should report in a '**Ready**' state.

After a few minutes when the machine is ready you should also be able to take a look at our new `Node` / `Machine` / `BareMetalHost` (take your pick!) in the console. Select '**Compute**' and then '**Nodes**' from the menu on the left and you should see all of our nodes along with the `Machine` that they correspond to:

<img src="img/worker-console.png"/>

If you click on the **"worker-2"** element on the left hand side you should be able to dive into the `Node` in a bit more detail and get some more statistics about it:

<img src="img/worker-overview.png"/>

That's it! We've successfully scaled our "baremetal" cluster using the baremetal operator.