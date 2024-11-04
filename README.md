# Kubernetes Grafana

![](https://i.imgur.com/waxVImv.png)
### [View all Roadmaps](https://github.com/nholuongut/all-roadmaps) &nbsp;&middot;&nbsp; [Best Practices](https://github.com/nholuongut/all-roadmaps/blob/main/public/best-practices/) &nbsp;&middot;&nbsp; [Questions](https://www.linkedin.com/in/nholuong/)
<br/>

This project is about running [Grafana](https://grafana.com/) on [Kubernetes](https://kubernetes.io/) with [Prometheus](https://prometheus.io/) as the datasource in a very opinionated and entirely declarative way. This allows easily operating Grafana highly available as if it was a stateless application - no need to run a clustered database for your dashboarding solution anymore!

Note that at this point this is primarily about getting into the same state as [kube-prometheus](https://github.com/nholuongut/prometheus-operator/tree/master/contrib/kube-prometheus) currently is. It is about packaging up Grafana as a reusable component, without dashboards. Dashboards are to be defined when using this Grafana package.

## What and why is happening here?

This repository exists because the Grafana stack in [kube-prometheus](https://github.com/nholuongut/prometheus-operator/tree/master/contrib/kube-prometheus) has gotten close to unmaintainable due to the many steps of generation and it's a very steep learning curve for newcomers.

Since Grafana v5, Grafana can be provisioned with dashboards from files. This project is primarily about generating a set of useful Grafana dashboards for use with and on Kubernetes using with Prometheus as the datasource.

In this repository everything is generated via jsonnet:

* The [Grafana dashboard sources configuration](https://github.com/nholuongut/kubernetes-grafana/blob/master/grafana/configs/dashboard-sources/dashboards.libsonnet).
* The Grafana dashboard datasource configuration, is part of the [configuration](https://github.com/nholuongut/kubernetes-grafana/blob/master/grafana/grafana.libsonnet#L17-L25), and is then simply [rendered to json](https://github.com/nholuongut/kubernetes-grafana/blob/master/grafana/grafana.libsonnet#L47).
* The Grafana dashboard definitions are defined as part of the [configuration](https://github.com/nholuongut/kubernetes-grafana/blob/master/grafana/grafana.libsonnet#L29).
* The [Grafana Kubernetes manifests](https://github.com/nholuongut/kubernetes-grafana/tree/master/grafana) with the help of [ksonnet/ksonnet-lib](https://github.com/ksonnet/ksonnet-lib).

With a single jsonnet command the whole stack is generated and can be applied against a Kubernetes cluster.

## Prerequisites

You need a running Kubernetes cluster in order to try this out, with the kube-prometheus stack deployed on it as have Docker installed to and be able to mount volumes correctly (this is **not** the case when using the Docker host of minikube).

For trying this out provision [minikube](https://github.com/nholuongut/minikube) with these settings:

```
minikube start --kubernetes-version=v1.9.3 --memory=4096 --bootstrapper=kubeadm --extra-config=kubelet.authentication-token-webhook=true --extra-config=kubelet.authorization-mode=Webhook --extra-config=scheduler.address=0.0.0.0 --extra-config=controller-manager.address=0.0.0.0
```

## Usage

Use this package in your own infrastructure using [`jsonnet-bundler`](https://github.com/nholuongut/jsonnet-bundler):

```
jb install github.com/nholuongut/kubernetes-grafana/grafana
```

An example of how to use it could be:

[embedmd]:# (examples/basic.jsonnet)
```jsonnet
local grafana = import 'grafana/grafana.libsonnet';

{
  _config:: {
    namespace: 'monitoring-grafana',
  },

  grafana: grafana($._config) + {
    service+: {
      spec+: {
        ports: [
          port {
            nodePort: 30910,
          }
          for port in super.ports
        ],
      },
    },
  },
}
```

This builds the entire Grafana stack with your own dashboards and a configurable namespace.

Simply run:

```
$ jsonnet -J vendor example.jsonnet
```

### Customizing

#### Adding dashboards

This setup is optimized to work best when Grafana is used declaratively, so when adding dashboards they are added declaratively as well. In jsonnet there are libraries available to avoid having to repeat boilerplate of Grafana dashboard json. An example with the [nholuongut/grafonnet-lib](https://github.com/nholuongut/grafonnet-lib):

[embedmd]:# (examples/dashboard-definition.jsonnet)
```jsonnet
local grafonnet = import 'github.com/nholuongut/grafonnet-lib/grafonnet/grafana.libsonnet';
local dashboard = grafonnet.dashboard;
local row = grafonnet.row;
local prometheus = grafonnet.prometheus;
local template = grafonnet.template;
local graphPanel = grafonnet.graphPanel;

local grafana = import 'grafana/grafana.libsonnet';

{
  _config:: {
    namespace: 'monitoring-grafana',
    dashboards+: {
      'my-dashboard.json':
        dashboard.new('My Dashboard')
        .addTemplate(
          {
            current: {
              text: 'Prometheus',
              value: 'Prometheus',
            },
            hide: 0,
            label: null,
            name: 'datasource',
            options: [],
            query: 'prometheus',
            refresh: 1,
            regex: '',
            type: 'datasource',
          },
        )
        .addRow(
          row.new()
          .addPanel(
            graphPanel.new('My Panel', span=6, datasource='$datasource')
            .addTarget(prometheus.target('vector(1)')),
          )
        ),
    },
  },

  grafana: grafana($._config) + {
    service+: {
      spec+: {
        ports: [
          port {
            nodePort: 30910,
          }
          for port in super.ports
        ],
      },
    },
  },
}
```

#### Organizing dashboards

If you have many dashboards and would like to organize them into folders, you can do that as well by specifying them in `folderDashboards` rather than `dashboards`.

[embedmd]:# (examples/dashboard-folder-definition.jsonnet)
```jsonnet
local grafonnet = import 'github.com/nholuongut/grafonnet-lib/grafonnet/grafana.libsonnet';
local dashboard = grafonnet.dashboard;
local row = grafonnet.row;
local prometheus = grafonnet.prometheus;
local template = grafonnet.template;
local graphPanel = grafonnet.graphPanel;

local grafana = import 'grafana/grafana.libsonnet';

{
  _config:: {
    namespace: 'monitoring-grafana',
    folderDashboards+: {
      Services: {
        'regional-services-dashboard.json': (import 'dashboards/regional-services-dashboard.json'),
        'global-services-dashboard.json': (import 'dashboards/global-services-dashboard.json'),
      },
      AWS: {
        'aws-ec2-dashboard.json': (import 'dashboards/aws-ec2-dashboard.json'),
        'aws-rds-dashboard.json': (import 'dashboards/aws-rds-dashboard.json'),
        'aws-sqs-dashboard.json': (import 'dashboards/aws-sqs-dashboard.json'),
      },
      ISTIO: {
        'istio-citadel-dashboard.json': (import 'dashboards/istio-citadel-dashboard.json'),
        'istio-galley-dashboard.json': (import 'dashboards/istio-galley-dashboard.json'),
        'istio-mesh-dashboard.json': (import 'dashboards/istio-mesh-dashboard.json'),
        'istio-pilot-dashboard.json': (import 'dashboards/istio-pilot-dashboard.json'),
      },
    },
  },

  grafana: grafana($._config) + {
    service+: {
      spec+: {
        ports: [
          port {
            nodePort: 30910,
          }
          for port in super.ports
        ],
      },
    },
  },
}
```

#### Dashboards mixins

Using the [kubernetes-mixin](https://github.com/nholuongut/kubernetes-mixin)s, simply install:

```
$ jb install github.com/nholuongut/kubernetes-mixin
```

And apply the mixin:

[embedmd]:# (examples/basic-with-mixin.jsonnet)
```jsonnet
local kubernetesMixin = import 'github.com/nholuongut/kubernetes-mixin/mixin.libsonnet';
local grafana = import 'grafana/grafana.libsonnet';

{
  _config:: {
    namespace: 'monitoring-grafana',
    dashboards: kubernetesMixin.grafanaDashboards,
  },

  grafana: grafana($._config) + {
    service+: {
      spec+: {
        ports: [
          port {
            nodePort: 30910,
          }
          for port in super.ports
        ],
      },
    },
  },
}
```

To generate, again simply run:

```
$ jsonnet -J vendor example-with-mixin.jsonnet
```

This yields a fully configured Grafana stack with useful Kubernetes dashboards.

#### Config customization

Grafana can be run with many different configurations. Different organizations have different preferences, therefore the Grafana configuration can be arbitrary modified. The configuration happens via the the `$._config.grafana.config` variable. The `$._config.grafana.config` field is compiled using jsonnet's `std.manifestIni` function. Additionally you can specify your organizations' LDAP configuration through `$._config.grafana.ldap` variable.

For example to modify Grafana configuration and set up LDAP use:

[embedmd]:# (examples/custom-ini.jsonnet)
```jsonnet
local grafana = import 'grafana/grafana.libsonnet';

{
  local customIni =
    grafana({
      _config+:: {
        namespace: 'monitoring-grafana',
        grafana+:: {
          config: {
            sections: {
              metrics: { enabled: true },
              'auth.ldap': {
                enabled: true,
                config_file: '/etc/grafana/ldap.toml',
                allow_sign_up: true,
              },
            },
          },
          ldap: |||
            [[servers]]
            host = "127.0.0.1"
            port = 389
            use_ssl = false
            start_tls = false
            ssl_skip_verify = false

            bind_dn = "cn=admin,dc=grafana,dc=org"
            bind_password = 'grafana'

            search_filter = "(cn=%s)"

            search_base_dns = ["dc=grafana,dc=org"]
          |||,
        },
      },
    }),

  apiVersion: 'v1',
  kind: 'List',
  items:
    customIni.dashboardDefinitions.items +
    [
      customIni.config,
      customIni.dashboardSources,
      customIni.dashboardDatasources,
      customIni.deployment,
      customIni.serviceAccount,
      customIni.service {
        spec+: { ports: [
          port {
            nodePort: 30910,
          }
          for port in super.ports
        ] },
      },
    ],
}
```

#### Plugins

The config object allows specifying an array of plugins to install at startup.

[embedmd]:# (examples/plugins.jsonnet)
```jsonnet
local grafana = import 'grafana/grafana.libsonnet';

{
  _config:: {
    namespace: 'monitoring-grafana',
    plugins: ['camptocamp-prometheus-alertmanager-datasource'],
  },

  grafana: grafana($._config) + {
    service+: {
      spec+: {
        ports: [
          port {
            nodePort: 30910,
          }
          for port in super.ports
        ],
      },
    },
  },
}
```

# Roadmap

There are a number of things missing for the Grafana stack and tooling to be fully migrated.

**If you are interested in working on any of these, please open a respective issue to avoid duplicating efforts.**

1. A tool to review Grafana dashboard changes on PRs. While reviewing jsonnet code is a lot easier than the large Grafana json sources, it's hard to imagine what that will actually end up looking like once rendered. Ideally a production-like environment is spun up and produces metrics to be graphed, then a tool could take a screenshot and [Grafana snapshot](http://docs.grafana.org/plugins/developing/snapshot-mode/) of the rendered Grafana dashboards. That way the changes can not only be reviewed in code but also visually. Similar to point 2 this should eventually be it's own project.

# 🚀 I'm are always open to your feedback.  Please contact as bellow information:
### [Contact ]
* [Name: Nho Luong]
* [Skype](luongutnho_skype)
* [Github](https://github.com/nholuongut/)
* [Linkedin](https://www.linkedin.com/in/nholuong/)
* [Email Address](luongutnho@hotmail.com)
* [PayPal.me](https://www.paypal.com/paypalme/nholuongut)

![](https://i.imgur.com/waxVImv.png)
![](Donate.png)
[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/nholuong)

# License
* Nho Luong (c). All Rights Reserved.🌟
