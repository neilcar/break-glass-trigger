# break-glass-trigger

This project combines Red Hat Advanced Cluster Security (ACS), Tekton Triggers, and Ansible to check whether an annotation used to bypass ACS deploy-time policies is valid.

ACS offers an option that allows devops teams to bypass deploy-time enforcement in the case of a 'break glass' emergency.  ACS will generate notifications when this is used and, if desired, this can be disabled entirely.  

It may be desirable, though, to keep this enabled but to check that a valid ticket has been created/approved/is open for the exception.

## Configuration

### Kubernetes

First, create a namespace for our project and ensure we're using it.

```
oc create ns breakglass
oc project breakglass
```

If needed, install [Tekton](https://tekton.dev/docs/getting-started/) and [Tekton Triggers](https://tekton.dev/docs/getting-started/).  Alternately, on OpenShift, install OpenShift Pipelines.

### Jira

In theory, you could create any number of approaches to managing these exceptions in Jira.  For simplicity, I created a new issue type ("Exception") and we're assuming that a valid exception is "In Progress".

The Ansible Jira module is going to need the base URL for your Jira server, a username, and a password.  I created an [API token](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/) to use for the password.  

With those values in hand, create our `jira-api-token` secret:

```
oc create secret generic jira-api-token \
  --from-literal=user=jira-user@neilcar.com \
  --from-literal=token=ThisIsTheAPIToken \
  --from-literal=uri=https://jira.neilcar.com
```

(replace all three of those values with your actual values, of course)

Create a project with the project key SEC for the security alerts when unauthorized deployments use a breakglass annotation.  For the sample, also create a project ABCD and create an exception, ABCD-1.

### Tekton

First, add the [Ansible Runner Tekton Task](https://hub.tekton.dev/tekton/task/ansible-runner) to our namespace.

```
oc apply --filename https://raw.githubusercontent.com/tektoncd/catalog/main/task/ansible-runner/0.1/ansible-runner.yaml
```

Next, apply the service account, trigger, and tasks/pipelines for our project:

```
oc apply --filename https://raw.githubusercontent.com/neilcar/break-glass-trigger/main/tekton/triggers/ansible-deployer-service-account.yaml
oc apply --filename https://raw.githubusercontent.com/neilcar/break-glass-trigger/main/tekton/triggers/breakglass-trigger.yaml
oc apply --filename https://raw.githubusercontent.com/neilcar/break-glass-trigger/main/tekton/pipelines/breakglass-ansible-pipeline-runner.yaml
```

### ACS

The definition for the trigger includes a Route for the webhook.  Find the route by running `oc get route` and create a new webhook integration with this value.

Add the integration to the "Emergency Annotation Used" 

## Testing

Right now, this assumes you have a demo environment.

To see a successful run, 

```
oc apply -f https://raw.githubusercontent.com/neilcar/break-glass-trigger/main/sample/visa-processor.yaml
```

(You may need to delete the existing deployment, first)

To see an unauthorized deployment,

```
oc apply -f https://raw.githubusercontent.com/neilcar/break-glass-trigger/main/sample/discover-processor.yaml
```

