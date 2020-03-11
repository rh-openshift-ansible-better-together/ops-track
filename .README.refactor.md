# OpenShift-Ansible Integration Lab

## 0 Introduction
Welcome to the Dev Track of the OpenShift + Ansible Better Together lab! Today you will learn about how Ansible can be leveraged to automate deployments and maintenance tasks on OpenShift. You'll gain experience around building an Ansible operator, and you'll integrate that operator with a data-driven app called WidgetFactory. Also, this lab will be performed on the brand new OpenShift 4 platform!

### 0.1 Scenario


 ##TODO: flush out scenario
You are the lead engineer of the OpenShift team and you have developer teams on the horizon ready to bombard your cluster. How do you create workspaces for them in a repeatable predictable fashion?

Spit balling:
- ldap config
- configure private registry
- status enforcement


 ##TODO: explain user env with backpack
## Welcome to backpack!
There are a few different tools you need in order to complete the lab:
- podman
- operator-sdk
- git
- oc

 ##TODO:
- usernames
- env ID
- login creds
- hostnames
- console/api access

### 1.1 Log in to OpenShift
You'll need to log into OpenShift with the `oc` tool to talk to the cluster from the command line. You'll also want to log into the UI.

See the table below for your location's cluster information.

 ##TODO: UDPATE
| Location | API Server | Web Console |
| -------- | ---------- | ----------- |
| Columbia, MD | https://api.cluster-columbia-c145.columbia-c145.open.redhat.com:6443 | http://console-openshift-console.apps.cluster-columbia-c145.columbia-c145.open.redhat.com |

Set an environment variable to reference your username and API Server:
```bash
export OCP_USER=<assigned-username> # For example, user60
export API_SERVER=<api-server>      # Referenced in the above table
```

Since you are the administrator of the cluster, you will be running as admin. This is necessary in order to have the permissions necessary for setting up cluster scoped operators.

Log into the UI by following your location's corresponding Web Console link from the table above. The login credentials are the same here as they were for `oc`.

### 1.2 Create OpenShift Project
You'll need to create an OpenShift namespace to deploy the operator into. Create a namespace with:
```bash
oc new-project managed-namespace-operator
```
## 2 Create Quay Account and Repositories

 ##TODO: Figure out if this is necessary. Maybe have to leverage internal registry to limit network traffic from pushing and pulling images
 ##TODONE: Validated ability to push to local registry

In this lab we're going to build a new operator with Ansible. The operator itself is simply a container image, so we need a place to store it so we can reference it in a future deployment.

Let's create a Quay account if you don't have one already. Go to https://quay.io/signin/ and click `Create Account` at the bottom. Provide your username, email address, and password. Optionally, you can sign in with an existing Google or GitHub account and follow the prompts, but you'll need to be sure to go to account settings and change your password since you'll need one to log in with docker.

Once your account is created, click on `Create New Repository` in the upper right. In the text box that says `Repository Name`, type `managed-namespace-operator`. Select the `Public` radio button. Then click `Create Public Repository`.

Create another repository called `managed-namespace-operator-test`, following the same procedure. This will be a heavier managed-namespace-operator that contains artifacts that are necessary for testing that we don't want as part of our production operator.

By the end of this section you should have two repositories in Quay under your account called `managed-namespace-operator` and `managed-namespace-operator-test`.

Set an environment variable to reference your Quay account:
```bash
export QUAY_USER=<quay-user>
printf "Quay Password: " ; read -sr QUAY_PASS_IN ; export QUAY_PASS=$QUAY_PASS_IN ; echo
```
## 3 Review the Ansible Operator

### 3.1 Operator Overview
The biggest concern from your team lead is that the team will get bogged down in managing tickets for creating namespaces for new development teams. The task at hand is to automate this process in a way that is trackable and helps enforce best practices for developers on the cluster. You decide to leverage the cool new [Operator framework](https://coreos.com/operators/) to provide simple way to create a way to create and update namespaces in a kubernetes native fashion.

An operator is an extention to the Kubernetes API. With an operator, we can create a 'ManagedNamespace' custom resource (CR), and OpenShift will be able to understand what we mean and create a new namespace with all of the proper metadata that your team needs for operations. In this case we'll also be able to set up the proper limits and quotas in order to make sure a single development team's application does not hog all of the cluster's resources.

### 3.2 Ansible Operator Structure
Navigate to the `managed-namespace-operator` directory:
```bash
cd $LAB/managed-namespace-operator
```
Here you will see the file structure of an Ansible operator. Check out the [operator-sdk](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md) Ansible documentation for a full overview of the Ansible operator. For this lab, here's what's important to know:

| File/Dir | Purpose |
| -------- | ------- |
| build/   | Contains the Dockerfile for building the Ansible operator |
| deploy/  | Contains the OpenShift resources necessary for deploying the Ansible operator and creating the ManagedNamespace CRD (custom resource definition) |
| roles/   | Contains the Ansible roles that the operator will be running when a CR (custom resource) is created |
| molecule/ | Contains the Ansible playbooks to perform [Molecule](https://github.com/ansible/molecule) testing on the Ansible operator |
| watches.yaml | Configures the operator to associate a CR to a particular Ansible role |

When the Ansible operator is deployed, it will listen for CRs and will apply the Ansible role accordingly. Operators are designed to maintain the "desired state", meaning it will run in a loop and will constantly re-run the roles in accordance to the CR spec to ensure that the desired state is always reached. Therefore, it's imperative that each role be written in an idempotant and stateless manner. It should also be able to handle any change to the OpenShift environment that may occur anywhere during role execution.

### 3.3 Review Ansible Roles
Let's dive a little deeper into the Ansible roles behind this operator. Find the `roles/` directory:
```bash
cd $LAB/managed-namespace-operator/roles
```
Here you'll find three Ansible roles:

| Role | Purpose |
| ---- | ------- |
| managed-namespace-operator | setup and update namespaces in OpenShift |


## 4 Write the Ansible Operator
Time to get a little more hands-on. We've left several placeholders throughout the operator for you to write some Ansible. Let's walk through the changes you'll have to make to allow the operator to be fully functional.

Each VM has the `vi` editor installed. We also provide the complete files under `$LAB/answers` for you to copy at the end of each section.

### 4.1 Finish the `managed-namespace-operator` Role
View the `main.yml` tasks file under the `mysql` role:
```bash
cat $LAB/managed-namespace-operator/roles/managed-namespace-operator/tasks/main.yml
```
Currently the the role is just a list of task names. We use these tasks to accomplish what we need to.

Under where it says `## TODO: Add module for creating namespace`, add the following line:
```yaml
- name: Create {{ namespace_name }} Namespace
```
This is the name of the first task of the `managed-namespace-operator` role. It makes the Ansible code more readable by letting developers know what the task is supposed to do, and it makes runtime output easier for administrators to understand in the event of troubleshooting.

Note also the `{{ namespace_name }}` string. This is a variable in Ansible, which is defined in `$LAB/managed-namespace-operator/roles/managed-namespace-operator/defaults/main.yml`. When expanded, it will equal the name of the namespace.

Let's add a couple more lines to the create namespace role, so that your task now looks like this:

```yaml
- name: Create Namespace
  k8s:
    state: present
    definition:
      kind: Namespace
      apiVersion: v1
      metadata:
        name: "{{ namespace_name }}"
      ##  labels:
      ##    size: "{{ size }}"

```

Notice the `k8s:` line. This tells Ansible to use the `k8s` module to perform an action on the OpenShift cluster. Think of a module as a function, in which `k8s:` is our "function" and `state:` and `definition` are the parameters to that function.

`state: present` tells the `k8s` module to create a resource to the cluster (as opposed to deleting it, which would instead be `state: absent`). `definition: ` tells the `k8s` module specifically what to create on the cluster. Let's add one more piece of code to complete this Ansible task to tie everything together. Add to the role so that your task now looks like this:

```yaml
- name: Create Resource Quota
  k8s:
    state: present
    definition: "{{ lookup('template', 'default-resourcequota.yml.j2' ) }}"

- name: Create Limit Range
  k8s:
    state: present
    definition: "{{ lookup('template', 'default-limitrange.yml.j2' ) }}"

```

Note how these tasks use the same module but instead use a lookup so that you can save configurations as files instead of inline. This promotes reusability of roles, and helps keep your environment logic seperate from your code. It also makes the role more readable. ##TODO: say more words


### 5.2 Build the Test Operator
We need to turn the Ansible plays into a Docker image so that it can be deployed and tested on OpenShift. We also need to make sure we include the test artifacts that are normally excluded from the production image. We can do this easily with the operator-sdk tool.

On the command line, navigate to the `managed-namespace-operator` directory and build the test operator:
```bash
cd $LAB/managed-namespace-operator
sed -i "s/BASEIMAGE/quay.io\/$QUAY_USER\/managed-namespace-operator/g" $LAB/managed-namespace-operator/build/test-framework/Dockerfile
operator-sdk build quay.io/$QUAY_USER/managed-namespace-operator --enable-tests
```
 ##TODO: Validate that enable-tests works cuz I couldn't get it to


Now that the test operator is built, let's push it to Quay with Docker.
```bash
docker login quay.io -u $QUAY_USER -p $QUAY_PASS
docker push quay.io/$QUAY_USER/managed-namespace-operator
```

You'll find that this is a somewhat large image. The production-sized operator is much smaller, which is why after we test and validate that the operator is working we should rebuild without the `--enable-tests` flag to remove the test artifacts.

### 5.3 Deploy the Test Operator
Now that the image has been built and is now in Quay, let's deploy it in your namespace. 

First, we need to create some resources to give the operator permission to edit your project. If you recall, the `deploy/` directory contains OpenShift resources that are required for the operator to work properly. It contains a service account, role, rolebindings, deployment, CRDs, and CRs. For now, let's create only what we need to test the operator:
```bash
cd $LAB/managed-namespace-operator
oc create -f deploy/service_account.yaml -n managed-namespace-operator
oc create -f deploy/role.yaml 
oc create -f deploy/role_binding.yaml 
```

## 6 Build and Deploy Production Operator
Now that we know the tests have passed, let's build the more lightweight production operator.

```bash
cd $LAB/managed-namespace-operator
operator-sdk build quay.io/$QUAY_USER/managed-namespace-operator
docker push quay.io/$QUAY_USER/managed-namespace-operator
sed -i "s/OPERATOR_IMAGE/quay.io\/$QUAY_USER\/managed-namespace-operator/g" $LAB/managed-namespace-operator/deploy/operator.yaml
oc create -f $LAB/managed-namespace-operator/deploy/operator.yaml
```

Wait for the pod to be ready

```bash
oc get pods -w
```

Now we can see the two containers that make up the ansible operator pod, the operator and the ansible runner First lets check out the operator container

```bash
oc logs <pod-name> -c operator
```

Notice that the operator is using the watch.yaml file to observe the OpenShift api for any actions on a 'ManagedNamespace' object. When it sees something, it then lets the ansible runner that it needs to run the designated role.

 ##TODO: Log snippet

Now lets take a look at the ansible continaer

```bash
oc logs <pod-name> -c ansible
```

Notice that there is not much going on right now. This is because we haven't given the operator anything to work with yet!







## 7 Create a Namespace
Now that the Ansible operator is deployed, it's super easy to add namespaces to OpenShift! First, let's check out the ManagedNamespace CR:
```bash
cat $LAB/managed-namespace-operator/deploy/crds/mysql/nekic_v1alpha1_initproject_cr.yaml
```

Notice that it has two spec fields, namespaceName and size. Right now, the operator is only cares about the namespaceName, since this will become the name of the namespace. We'll focus on the size later.



Let's create the resource with:
```bash
oc create -f $LAB/managed-namespace-operator/deploy/crds/mysql/nekic_v1alpha1_initproject_cr.yaml

You should get a message saying that the ManagedNamespace resource was created. The new namepsce will get added pretty quickly - right now the operator pod running the corresponding Ansible role. We can see this role in action by checking out the operator logs:
```bash
oc logs --follow $(oc get po | grep managed-namespace-operator | awk '{print $1}')
```

When the role is finished, you should see something like `ansible-runner exited successfully` in the logs, as well as a new namespace added to the cluster. This is pretty slick and all but we all know that one development team that will need more resources. Let's add the concept of t-shirt sizes in order to make our lives easier down the road. 







## 8 Add T-Shirt sizes
To accomplish this, we will need to update some of the logic in our ansible role. Uncomment the labels section
```yaml
- name: Create {{ namespace_name }} Namespace
  k8s:
    state: present
    definition:
      kind: Namespace
      apiVersion: v1
      metadata:
        labels:
          size: "{{ size }}"         
        name: "{{ namespace_name }}"
```

Now when this task is called, the k8s module will ensure that this label is added to each of the managed namespaces. This will make auditing and monitoring easier since an administrator see this label and understand the amount of reasources a namespace should be allocated. It also makes forecasting resource consumption simpler with codified t-shirt sizes

Next, we need to update the quota and limit logic to select the proper size t-shirt template instead of the default size. To accomplish this, update the lookup line to include the size parameter that gets passed in from the ManagedNamespace cluster resource object. It should look like this:

```yaml
- name: Create Resource Quota
  k8s:
    state: present
    definition: "{{ lookup('template', '{{ size }}-resourcequota.yml.j2' ) }}"

- name: Create Limit Range
  k8s:
    state: present
    definition: "{{ lookup('template', '{{ size }}-limitrange.yml.j2' ) }}"
```

You can take a look at the template directory also within this role and see that it is seeded with some basic t-shirt sizes.

```yaml
take a look at medium
```

Notice that name of the resource is generic, but it is labeled the proper size. This will help us down the road in the event you want to upgrade a namespace to a larger size. Instead of having to deal with deleting one quota and adding another, you can patch or apply the updated quota and Openshift will take care of the merging logic. This avoids any lapses in quota management.

With the role updated, rebuild the image and push it up to the registry. This will make it available for Openshift to deploy it onto the cluster.
```bash
operator-sdk build quay.io/$QUAY_USER/managed-namespace-operator
docker push quay.io/$QUAY_USER/managed-namespace-operator
```

With the new image available, trigger a new deployment so that OpenShift will rollout the new image. 

##TODO: figure out imagestreams

```bash
oc deploy dc/managed-namespace-operator
```

Watch the rollout for the new pod to become ready

```bash
oc get pods -w
```

Now the operator is running your new ansible role that can handle t-shirt sizes. Let's try creating a new namespace with a medium t-shirt size. 
```bash
oc create -f $LAB/managed-namespace-operator/deploy/crds/medium-namespace.yaml
```

Watch for the new namespace to be created. Be fast!
```bash
oc get namespaces -w
```

Once it's created, check out its quotas.
```bash
oc get quotas -n medium-namespaces -o yaml

 ##TODO: put in quota code block

Notice how the label is set to medium and the limits are higher!




## 9 Updating existing namespaces

Good news! The development team that you created the first namespace for got approval to ramp up their deployments on OpenShift. This means that their namespaces is going to need more resources. How can this be done?

Even better news! The managed-namespace-operator can already handle this! All you need to do is update the ManagedNamespace CR on the cluster to tell the operator to update the namespace. Update the label on the test-project CR to now be set to 'large'
```bash
oc edit ManagedNamespace.nekic.io example-init-project
```

Once you save that, the operator will go ahead to update the quota


 ##TODO: maybe have user set up a watch on the quota in test-project namespace in order to show update











What if we deploy bad stuff on to cluster, and the operator has to figure out how to clean it up


 ##TODO: High Level Lab Flow
- Edit operator
	- add section for creating namespace
- Testing? ## Not sure what would need to be done for this
- Build operator image
- Push operator image to internal registry
- Deploy operator
- App needs bigger quota
- Need to be able to resize
- Update label on namespace task
- Update template lookup to have size option for quota and limits
- Rebuild operator image
- Push operator image to internal registry
- Create a new ManagedNamespace CR
- Validate namespace
- Update existing ManagedNamespace CR
- Watch to see it get updated
- Mention GitOps



Verification steps:
copy work from agnosticd/ansible/configs/ocp4-workshop/post_software.yml post-flight-check section
- Validate that there are no defaultProject requests
- Internal Registry pod is up
- get internal registry service
- get user token
- Login into internal registry
- Create post-flight project
- build and push image to internal registry
- deploy operator
- wait for operator pod to run
- create managednamespace object
- check resulting project status
- clean up operator
  - namespace
  - clusterresource
  - crd
  - clusterroles
  - clusterrolebinding
- delete the test-project
- make sure image is deleted from internal registry




- pull the managed-namespace-operator into the backpack
- set up internal registry as insecure???

Things To figure out:
- myvars.yaml
- post branch onto github
- define workloads to be run
  - copy work from agnosticd/ansible/configs/ocp4-workshop/post_software.yml post-flight-check section




Creating operator notes
```bash
  operator-sdk new managed-namespace-operator --api-version=beter.together.io/v1alpha1 --kind=ManagedNamespace --type=ansible
```

update WATCH_NAMESPACE
```bash
vi deploy/operator.yaml
```

```yaml
          env:
            - name: WATCH_NAMESPACE
              value: ""
```

update role to cluster role
```bash
mv deploy/role.yaml deploy/cluster_role.yaml
```
```bash
vi deploy/cluster_role.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole ###UPDATE HERE
metadata:
  creationTimestamp: null
  name: managed-namespace-operator
```

add required rules to ClusterRole ### Note this is adding an additional item to the list of roles
```bash
vi deploy/cluster_role.yalm
```

```yaml
- apiGroups:
  - ""
  resources:
  - namespaces
  - resourcequotas
  - limitranges
  verbs:
  - "*"
```


update rolebinding to clusterrolebinding
```bash
mv deploy/role_binding.yaml deploy/cluster_role_binding.yaml
```

```yaml
kind: ClusterRoleBinding  ## UPDATED
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: managed-namespace-operator
subjects:
- kind: ServiceAccount
  name: managed-namespace-operator
  namespace: managed-namespace-operator   ## ADDED LINE
roleRef:
  kind: ClusterRole  ## UPDATED
  name: managed-namespace-operator
  apiGroup: rbac.authorization.k8s.io


update crd to be cluster scoped
```bash
vi deploy/crds/better_v1alpha1_managednamespace_crd.yaml
```

```yaml
metadata:
  name: managednamespaces.better.together.io
spec:
  group: better.together.io
  names:
    kind: ManagedNamespace
    listKind: ManagedNamespaceList
    plural: managednamespaces
    singular: managednamespace
  scope: Cluster ## UPDATED
...
```
