---
layout: bt_wiki
title: Manager Policies
category: Manager
draft: false
abstract: "A guide to Cloudify Policies"
weight: 70
aliases: /manager_policies/overview/

types_yaml_link: http://www.getcloudify.org/spec/cloudify/4.3/types.yaml
diamond_plugin_ref: plugin-diamond.html
dsl_groups_spec: dsl-spec-groups.html
workflows_dsl_spec: dsl-spec-workflows.html
diamond_package_ref: https://github.com/BrightcoveOS/Diamond
---

# Overview

Policies provide a way of analyzing a stream of events that correspond to a group of nodes (and their instances).
The analysis process occurs in real time and enables actions to be triggered, based on its outcome.

The stream of events for each node instance is read from a deployment-specific RabbitMQ queue by the policy engine ([Riemann](http://riemann.io/)). It can come from any source but is typically generated by a monitoring agent installed on the node's host. (See: [Diamond Plugin]({{< relref "working_with/official_plugins/Configuration/diamond.md" >}})).

The analysis is performed using [Riemann](http://riemann.io/). Cloudify starts a Riemann core for each deployment with its own Riemann configuration. The Riemann configuration is generated, based on the Blueprint's `groups` section that will be described later in this guide.

{{% note title="Note" %}}
The Cloudify policy engine currently uses Riemann. This component is subject to change in the future.
{{% /note %}}

Part of the API that Cloudify provides on top of Riemann, enables policy triggers to be activated. Usually, that entails executing a workflow in response to a state--for example, if the tomcat CPU usage was above 90% for the last five minutes.
The workflow might increase the number of node instances, for example, to handle an increased load.

The logic fo the policies is written in [Clojure](http://clojure.org/) using Riemann's API. Cloudify adds a thin API layer on top of these APIs.

The [Nodecellar Example](https://github.com/cloudify-examples/nodecellar-auto-scale-auto-heal-blueprint) is a great example how to apply Cloudify policies.

# Using Policies

Policies are configured in the `groups` section of a Blueprint, as shown in the following example.

{{< highlight  yaml  >}}
groups:
  # arbitrary group name
  my_group:

    # All nodes this group applies to
    # in this case, assume node_vm and mongo_vm
    # were defined in the node_templates section.
    # All events that belong to node instances of
    # these nodes are processed by all policies specified
    # within the group.
    members: [node_vm, mongo_vm]

    # Each group specifies a set of policies
    policies:

      # arbitrary policy name
      my_policy:

        # The policy type to use. Here a built-in 
        # policy type that identifies host failures 
        # based on a keep alive mechanism is used
        type: cloudify.policies.types.host_failure

        # policy specific configuration
        properties:
          # This is not really a host_failure property, it only serves as an example.
          some_prop: some_value

        # This section specifies what will be
        # triggered when the policy is "triggered"
        # (more than one trigger can be specified)
        triggers:
          # arbitrary trigger name
          execute_heal_workflow:

            # Using the built-in 'execute_workflow' trigger
            type: cloudify.policies.triggers.execute_workflow

            # Trigger-specific configuration
            parameters:
              workflow: heal
              workflow_parameters:
                # The failing node instance ID is exposed by the
                # host_failure policy on the event that triggered the
                # execute workflow trigger. Access to properties on
                # that event are as demonstrated below.
                node_instance_id: { get_property: [SELF, node_id] }

{{< /highlight >}}

# Built-in Policies

Cloudify includes a number of built-in policies that are declared in [`types.yaml`]({{< field "types_yaml_link" >}}), which is usually imported either directly or indirectly via other imports. See [Built-in Policies]({{< relref "developer/manager_policies/built-in-policies.md" >}}) for more information.

# Writing a Custom Policy

If you are an advanced user, you might want to create custom policies. For more information, see [policies authoring guide]({{< relref "developer/manager_policies/creating-policies.md" >}}).

# Auto Healing

Auto healing is the process of automatically identifying when a certain component of your system is failing and fixing the system state without any user intervention. Cloudify includes built-in support for auto healing different components.

{{% warning title="Limitation" %}}
Currently, only compute nodes can have auto healing configured for them. Otherwise, when a certain compute node instance fails, node instances that are contained in it are also likely to fail, which in turn can cause the heal workflow to be triggered multiple times.
{{% /warning %}}

There are several items that must be configured for auto healing to work. The process is described in this topic. The process involves:

* Configuring monitoring on the compute nodes that will be subject to auto healing
* Configuring the `groups` section with appropriate policy types and triggers

## Monitoring

You must add monitoring to the compute nodes that require auto healing to be enabled.

This example uses the built-in [Diamond]({{< field "diamond_package_ref" >}}) support that is bundled with Cloudify. See the [Diamond Plugin]({{< relref "working_with/official_plugins/Configuration/diamond.md" >}}) documentation for more about configuring Diamond-based monitoring.

{{< highlight  yaml  >}}
node_templates:
  some_vm:
    interfaces:
      cloudify.interfaces.monitoring:
        start:
          implementation: diamond.diamond_agent.tasks.add_collectors
          inputs:
            collectors_config:
              ExampleCollector: {}
        stop:
          implementation: diamond.diamond_agent.tasks.del_collectors
          inputs:
            collectors_config:
              ExampleCollector: {}
{{< /highlight >}}

The `ExampleCollector` generates a single metric that constantly has the value `42`. This collector is used as a "heartbeat" collector, together with the `host_failure` policy described below.

## Workflow Configuration

The `heal` [workflow]({{< relref "developer/blueprints/spec-workflows.md" >}}) executes a sequence of tasks that is similar to calling the `uninstall` workflow, followed by the `install` workflow. The main difference is that the `heal` workflow operates on the subset of node instances that are contained within the failing node instance's compute, and on the relationships these node instances have with other node instances. The workflow reinstalls the entire compute that contains the failing node and handles all appropriate relationships between the node instances inside the compute and the other node instances.

## Groups Configuration

After monitoring is configured, [groups]({{< relref "developer/blueprints/spec-groups.md" >}}) must be configured. The following example contains a number of inline comments. It is recommended that you read them, to ensure that you have a good understanding of the process.

{{< highlight  yaml  >}}
groups:
  some_vm_group:
    # Adding the some_vm node template that was previously configured
    members: [some_vm]
    policies:
      host_failure_policy:
        # Using the 'host_failure' policy type
        type: cloudify.policies.host_failure

        # Name of the service we want to shortlist (using regular expressions) and
        # watch - every Diamond event has the service field set to some value.
        # In our case, the ExampleCollector sends events with this value set to "example".
        properties:
          service:
            - example

        triggers:
          heal_trigger:
            # Using the 'execute_workflow' policy trigger
            type: cloudify.policies.triggers.execute_workflow
            parameters:
              # Configuring this trigger to execute the heal workflow
              workflow: heal

              # The heal workflow gets its parameters 
              # from the event that triggered its execution              
              workflow_parameters:
                # 'node_id' is the node instance ID
                # of the node that failed. In this example it is
                # something similar to 'some_vm_afd34'
                node_instance_id: { get_property: [SELF, node_id] }

                # Contextual information added by the triggering policy
                diagnose_value: { get_property: [SELF, diagnose] }
{{< /highlight >}}

In this example, a group was configured that consists of the `some_vm` node specified earlier. Then, a single `host_failure` policy was configured for the group. The policy has one `execute_workflow` policy trigger configured, which is mapped to execute the `heal` workflow.




