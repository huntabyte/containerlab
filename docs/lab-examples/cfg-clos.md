|                               |                                                                                |
| ----------------------------- | ------------------------------------------------------------------------------ |
| **Description**               | A 5-stage Clos topology based on SR Linux nodes configured using Config Engine |
| **Components**                | [Nokia SR Linux][srl], [Nokia SR OS][sros]                                     |
| **Resource requirements**[^1] | :fontawesome-solid-microchip: 4 <br/>:fontawesome-solid-memory: 12 GB          |
| **Lab folder**                | [lab-examples/Clos03][labfolder]                                               |
| **Version information**       | `containerlab:0.19.0`, `srlinux:21.6.2-67`, `vr-sros:21.7.R1`                  |
| **Authors**                   | Bastien Claeys                                                                 |

## Description
This lab features a 5-stage (two tier) Clos fabric which consists of the following components:

* leaf and spine elements based on Nokia SR Linux
* DC Gateway elements running SR OS
* CE devices emulated by a single SR OS device to allow for BGP configuration between a Leaf and a CE. 

In addition to the [topology file][topofile], the lab directory contains a set of files to handle the interface and BGP configurations, providing an environment ready for the provisioning of services/workloads.​

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:0,&quot;zoom&quot;:1.5,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://raw.githubusercontent.com/srl-labs/containerlab/diagrams/topo_cfg_clos01.drawio&quot;}"></div>


This topology leverages the Configuration Engine embedded in Containerlab. With the provided templates, configuration of nodes can be achieved in a few seconds.

The provided topology is using the following configuration:

* Underlay networking is achieved via eBGP
* Overlay iBGP sessions established to exchange EVPN routes

## Lab Walkthrough
### Execution
```bash
# Deploy the topology
$ containerlab deploy -t cfg-clos.clab.yml

# Generate and apply the configuration from the templates
$ containerlab config -t cfg-clos.clab.yml  -p . -l cfg-clos
```

### Understanding the Configuration Engine

The [Configuration Engine][cfgengine] of ContainerLab allows to prepare configuration templates, such that adding a new node in a topology requires only a little effort. The following steps will guide you through the files and their execution, to help understand the process behind it.

#### a) Declaring variables

This topology contains multiple nodes, each one having its own specific aspects. Generating a different configuration for all of them at once seems a bit tricky. But with the usage of variables within the topology file, the Configuration Engine can easily customize templates for each device. Let's dissect the topology file.

Several variable types are used in the topology to flexibly configure the nodes: global variables, node variables and link variables.

The variables are scoped under `.vars` container which is present on all of the above mentioned levels.

##### Global variables
```yaml
topology:
  defaults:
    config:
        vars:
        overlay_as: 65555
```

##### Node variables
```yaml
topology:
  nodes:
    dcgw1:
      kind: vr-sros
      type: sr-1
      config:
        vars:
          system_ip: 10.0.0.31
          as: 65030
```

##### Link variables
```yaml
topology:
  links:
    - endpoints: ["dcgw1:eth1","spine1:e1-31"]
      vars:
        port: [1/1/c1, ethernet-1/31]
        clab_link_ip: 100.31.21.1/30
        bgp_underlay: true
```

Those defined variables are declared in the topology and then referenced directly in the templates.
Note the usage of a [magic variable][magic] in the link context - `clab_ip_link`.

#### b) Generating variables
Once the topology is defined and variables are placed in the relevant levels, the full list of variables can be retrieved and introspected: 
```
$ containerlab config --topo cfg-clos.clab.yml template --vars
```

The following output represents the variables generated for `dcgw1` node:
```
INFO[0000] dcgw1 vars = as: 65030
clab_links:
- bgp_underlay: true
  clab_far:
    bgp_underlay: true
    clab_link_ip: 100.31.22.2/30
    clab_link_name: to_dcgw1
    clab_node: spine2
    port: ethernet-1/31
  clab_link_ip: 100.31.22.1/30
  clab_link_name: to_spine2
  port: 1/1/c2
- bgp_underlay: true
  clab_far:
    bgp_underlay: true
    clab_link_ip: 100.31.21.2/30
    clab_link_name: to_dcgw1
    clab_node: spine1
    port: ethernet-1/31
  clab_link_ip: 100.31.21.1/30
  clab_link_name: to_spine1
  port: 1/1/c1
clab_node: dcgw1
clab_nodes: '{leaf1: {...}, dcgw1: {...}, leaf4: {...}, spine2: {...}, spine1: {...},
  leaf2: {...}, leaf3: {...}, sros-client: {...}, dcgw2: {...}, }'
clab_role: vr-sros
overlay_as: 65555
system_ip: 10.0.0.31
```

This output is extremely helpful for a template designer, as it shows which variables and their values can be used in the configuration template.

#### c) Writing templates
Now that we have defined a topology and verified that output variables were correct, let's see how to use them in a template.

This topology contains of leaves, spines, DCGWs and CE elements. Even though we have many nodes in the topology, as far as the node kinds are concerned, we only have two: srlinux and sros.

Understandably, the configuration of SR Linux nodes differs from SR OS one, hence we will create two templates, one for SR Linux nodes and one for SR OS nodes.

Containerlab config engine by default assumes that templates are created per containerlab kind, in our case, the kinds are: `srl` and `vr-sros`.

So what we will do here is create two templates named `cfg-clos__srl.tmpl` and `cfg-clos_vr-sros.tmpl`. That way config engine will know which template to use against which node.

!!!note
    Notice, how node's kind is encoded in the template name by suffixing the file name with `__$kindName`.

The below section of `cfg-clos__srl.tmpl` template illustrates how each set of variables can be used to generate node's configuration.

```
{{/* If the bgp_underlay flag specified under the link then configure underlay ebgp on links */}}
{{- range $name, $link := .clab_links -}}
  {{- if .bgp_underlay }}
/ network-instance default protocols bgp neighbor {{ ip $link.clab_far.clab_link_ip }}  peer-group underlay
/ network-instance default protocols bgp neighbor {{ ip $link.clab_far.clab_link_ip }} peer-as {{(index $.clab_nodes $link.clab_far.clab_node).as}}
  {{- end }} 
{{- end -}}
```

`clab_links` contains all the links related to a node. `range` iterates on that variable and for each link, the existence of `bgp_underlay` variable is checked. If so, a peering is defined using the remote link IP address and AS number.

Feel free to navigate through the templates, they will teach you how useful variables can be in this context.

#### d) Generating configurations from templates
Now that we have seen how variables are used, let's see the resulting configuration with :
```
containerlab config -t cfg-clos.clab.yml template -p . -l cfg-clos
```

This command will dump the final configuration as it would be sent to the node in a later step. Make sure that the config appears to be correct before proceeding.

#### e) Applying the configurations
To apply the templated configuration on the deployed nodes, simply use :
```
containerlab config -t cfg-clos.clab.yml -p . -l cfg-clos
```

Containerlab will render the templates and use SSH client to connect to the nodes and apply the configuration.

!!!note
    Entering in the configuration mode and commit steps are carried out by containerlab and are not part of the templates.

[srl]: https://www.nokia.com/networks/products/service-router-linux-NOS/
[sros]: https://www.nokia.com/networks/products/service-router-operating-system/
[topofile]: https://github.com/srl-labs/containerlab/tree/main/lab-examples/clos03/cfg-clos.clab.yml
[labfolder]: https://github.com/srl-labs/containerlab/tree/main/lab-examples/clos03
[cfgengine]: https://github.com/hellt/clab-config-demo
[magic]: https://github.com/hellt/clab-config-demo#5-magic-variables

[^1]: Resource requirements are provisional. Consult with SR Linux Software Installation guide for additional information.

<script type="text/javascript" src="https://cdn.jsdelivr.net/gh/hellt/drawio-js@main/embed2.js" async></script>