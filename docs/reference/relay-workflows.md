# Workflow reference

Use a Relay workflow to define the distinct steps in the task you're automating.

Relay workflows use four top-level maps: `description`, `parameters`, `steps`, and `triggers`. Of these, only `steps` is required.

## Description

Start your workflow with the `description` key. For example:

```yaml
description: This workflow prints out a hello message to the logs. It also runs some simple terminal commands and prints their results to the logs.
```

## Parameters

Parameters enable Relay to provide outside data into the workflow as it runs. You can optionally provide a default value for each parameter, which a user-specified value will override at runtime. If you do not provide a default and do not supply a value, the workflow run will fail.

Relay currently only supports parameters whose values are strings.

```yaml
parameters:
  myparam:
    description: "A typical user supplied parameter"
    default: defaultvalue
```

## Steps

The `steps` key makes up the body of your workflow. It contains an array of steps, where each step must consist of a `name`, `image`, and `spec`, plus an optional `dependsOn` key.

### name

(string) The name of the step. Must be unique across the workflow.

```yaml
name: echo
```

### image

(string) The container image and tag you're using for the step. Relay's default registry is [Docker hub](https://hub.docker.com), so if the image is hosted there you can use the short `imagename:tag` syntax. Otherwise, use the long form like `gcr.io/account/image:tag`

```yaml
image: alpine:latest
```

### spec

(array) Short for 'specification'; this section provides context that's specific to the step. The contents of `spec` will depend on the implementation of the task container; programmers may find it useful to think of it as providing values to a function. For example, a step which uses the [relaysh/jira-server-step-issue-transition](https://hub.docker.com/r/relaysh/jira-server-step-issue-transition) container to close a Jira ticket requires `url` and `issue` keys in its `spec`. For a list of step containers curated by Puppet see [step containers](../step-specifications.md).

### dependsOn

(string or array of strings) As the name suggests, `dependsOn` indicates that this step depends on another step in the workflow. Each value must be a valid `name` attribute for another step. This key is useful if you need to set an explicit sequential order for your steps. Without `dependsOn` or implicit ordering requirements (see the [!Output type](../reference/relay-types.md)), Relay will run your steps in parallel to speed up execution.

```yaml
dependsOn:
- deploy_test_cluster
```

### input

(array of strings) If the container is configured to accept it, you can use the `input` map to provide a list of commands that Relay passes to the entrypoint.

### inputFile

(string) If the container is configured to accept it, you can use the `inputFile` attribute to supply a URL to a file which Relay will download and provide as input to the entrypoint. The URL must be accessible to the public internet.

## Triggers

Triggers map incoming events to workflows. The `triggers` key contains an array of individual triggers, any of which can cause the workflow to run. Each trigger must have `name` and `source` keys, and may optionally have `binding` and `when` keys.

### name

(string) A name for this trigger; must be unique to the workflow.

### source

(map) The source for the trigger is a description of the event which will cause the workflow to run. There are currently three kinds of sources available; `source` requires a `type` attribute to indicate which one to use, and each type has additional required fields.

#### schedule

A Schedule trigger type works, as the name suggests, on a time-based schedule. It needs a key named `schedule` whose value is a string in  the standard Unix 'cron' syntax. There's a handy [crontab syntax generator](https://crontab.guru/) that can help you build a schedule if you're not familiar with cron.

```yaml
triggers:
 - name: my-schedule-trigger
   source:
     type: schedule
     schedule: '*/5 * * * *'   # runs every 5 minutes
```

#### webhook

A Webhook trigger matches an incoming webhook payload from an external service to the workflow. Relay runs a container image in reponse to the webhook payload, so it requires an `image` field that is a reference to a container image hosted on a registry, same as the `image` field in a `step` definition.

```yaml
triggers:
  - name: my-webhook-trigger
    source:
      type: webhook
      image: relaysh/dockerhub-trigger-image-pushed
```

This example runs the `dockerhub-trigger-image-pushed` container in response to receiving a webhook. Once you add a webhook trigger to a workflow, the Relay app will prompt you to complete the webhook configuration in the sidebar. For information on building your own webhook containers, see the [Integrating with Relay](../integrating-with-relay.md) documentation.

#### push

A push trigger uses events that come into Relay through its API to trigger workflows. Unlike webhook triggers, push triggers do not require a container image because they are handled by the Relay service itself. If you have an external system which can generate a JSON payload but doesn't support webhooks, push triggers are a good option.

To use them, add a push trigger definition to your workflow and view it on the Relay web app to generate a trigger-specific token. This token is scoped only to the workflow which uses the trigger, which makes it more secure than providing your login token. The external system should make a POST request to `https://api.relay.sh/api/events` with a JSON payload in a map called `data`. The workflow then uses the `!Data` function to extract keys from the map and bind them to workflow parameters. 

For example, a workflow containing a push trigger would look like this:

```yaml
triggers:
  - name: my-push-trigger
    source:
      type: push
    binding:
      parameters:
        message: !Data eventmessage
parameters:
  message:
    description: A string to display
    default: Default message, override me
steps:
  - name: display-message
    image: relaysh/core
    spec:
      message: !Parameter message
    input:
      - echo "My message was $(ni get {.message})"
```

This workflow will be triggered by an HTTP request such as this curl command:

```shell
export TOKEN=... # set this from the web app
curl -X POST -H "Authorization: Bearer $TOKEN" \
   -d '{"data": {"eventmessage": "This is a push event"}}' \
   https://api.relay.sh/api/events
```

### binding

The `binding` key of a trigger definition maps incoming data from the event to parameters in the workflow. This allows you, for example, extract a field from the json payload of a webhook. A `binding` has one field, `parameters`, which is a map whose keys must match the names of parameters inside the workflow. The values can use [Functions](./relay-functions.md) or [Data types](./relay-types.md) in order to extract and manipulate data into the form the workflow parameter expects.

The `!Data` custom type is particularly helpful here, because it will be populated with the keys from an incoming `webhook` or `push` event.

```yaml
triggers:
  - name: my-webhook-trigger
    source:
      type: webhook
      image: relaysh/dockerhub-trigger-image-pushed
    binding:
      parameters:
        dockerTagName: !Data tag
parameters:
  dockerTagName:
    default: latest
```

 Expanding on the Docker hub example, this trigger extracts the `tag` field from the incoming webhook and maps it to the parameter `dockerTagName`, which is used in the workflow to override the default value of `latest`.

 ### when

 The same syntax that Relay uses to provide conditional execution of steps is also available for triggers. This uses the `when` key, whose value is an expression that must evaluate to "true" in order for the workflow to run. See the [Conditional execution](../conditionals.md) docs for more details on the syntax and usage.

Similar to the `binding` map, it's handy to make decisions about execution based on the contents of the incoming event. The `!Data` custom type allows you to extract fields from webhooks and push events as a basis for comparison.

```yaml
triggers:
  - name: conditional-trigger
    source:
      type: webhook
      image: relaysh/core
    when: !Fn.equals[!Data environment, 'production']
```

The [Relay functions](./relay-functions.md) reference has more examples of the `!Fn.*` syntax.