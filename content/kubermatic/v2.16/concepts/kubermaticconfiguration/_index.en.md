+++
title = "KubermaticConfiguration"
date = 2020-02-04T12:07:15+02:00
weight = 20

+++

## Overview

The KubermaticConfiguration CustomResourceDefinition is used for configuring the Kubermatic Kubernetes Platform (KKP) Operator and
replaces what previously was done with the `values.yaml` for the KKP Helm chart.

The following is an example configuration, showing all possible options. Note that all fields that you
don't define explicitly are always defaulted to these values.

```yaml
{{< readfile "kubermatic/v2.16/data/kubermaticConfiguration.yaml" >}}
```
