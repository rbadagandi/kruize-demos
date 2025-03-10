# Kruize Demo Scripts

Want to try out Kruize? You've come to the right place!

## What is Kruize HPOaaS?

Machine learning is a process of teaching a system to make accurate predictions based on the data fed. Hyperparamter optimization (/ tuning) helps to choose the right set of parameters for a learning algorithm. HPO uses different methods like Manual, Random search, Grid search, Bayesian optimization. [Kruize HPOaaS](https://github.com/kruize/hpo/blob/master/README.md) currently uses Bayesian optimization because of the multiple advantages that it provides.

### [HPOaaS demo](/hpo_demo_setup.sh)

#### Goal
  The user has an objective function that needs to be either maximized or minimized. They also have a search space (aka Domain space) which consists of a group of hyperparameters and the range within which they operate. The user provides the objective function and the search space to HPOaaS (running natively or on minikube). HPOaaS then provides trial configs which the user can test (Eg with a benchmark) and then return the result back to HPOaaS. This is then done in a loop until the entire gets a satisfactory result with the objective function.
#### Steps
  This demo starts HPOaaS natively and then starts the TFB benchmark on minikube. It then starts an experiment with the given search space [JSON](/hpo_helpers/search_space.json) and runs it for 3 trials.
#### What does it do?
  Currently it provides a log that consists the results of all the trials and needs to be manually evaluated to see which configuration provided the best result.
#### pre-req
  It expects minikube to be installed with atleast 8 CPUs and 16384MB Memory.
#### Customize it for your usecase
   Follow [Customize the script for your usecase](hpo_helpers/README.md) to customize the demo script for your usecase.
##### WARNING: The script deletes any existing minikube cluster.

## What is Kruize Autotune?

[Kruize Autotune](https://github.com/kruize/autotune/blob/master/README.md) is an Autonomous Performance Tuning Tool for Kubernetes. Autotune accepts a user provided Service Level Objective or "slo" goal to optimize application performance. It uses Prometheus to identify "layers" of an application that it is monitoring and matches tunables from those layers to the user provided slo. It then runs experiments with the help of a hyperparameter optimization framework to arrive at the most optimal values for the identified tunables to get a better result for the user provided slo.

Autotune can take an arbitrarily large set of tunables and run experiments to continually optimize the user provided slo in incremental steps. For this reason, it does not necessarily have a "best" value for a set of tunables, only a "better" one than what is currently deployed.

### [Autotune minikube demo](/autotune_minikube_demo_setup.sh)

#### Goal
The user has an application that is deployed to minikube and is looking to improve some aspect of performance of the application. The user specifies an "objective function" in an "Autotune object" that defines the performance aspect of the application that needs to be optimized. Autotune, also deployed to minikube, then analyzes the user application, breaks it down into its component layers and provides tunables associated with each layer that can help optimize the user provided objective function.

#### Steps
This demo installs Autotune along with Prometheus and Grafana to minikube. It also deploys two example REST CRUD applications, quakus galaxies and springboot petclinic, to the minikube cluster. It then deploys the "Autotune Objects" that define the objective function of the performance tuning that needs to be done for each application.

#### What does it do?
  It provides a list of URLs that defines the tunables for a user provided slo. See the docs for the definition of the [REST API](https://github.com/kruize/autotune/blob/master/design/API.md) associated with these URLs.

  ```
  Info: Access Autotune tunables at http://192.168.39.138:30110/listAutotuneTunables
  Info: Autotune is monitoring these application stacks http://192.168.39.138:30110/listStacks
  Info: List Layers in application stacks that Autotune is monitoring http://192.168.39.138:30110/listStackLayers
  Info: List Tunables in application stacks that Autotune is monitoring http://192.168.39.138:30110/listStackTunables
  Info: Autotune searchSpace at http://192.168.39.138:30110/searchSpace

  Info: Access autotune objects using: kubectl -n default get autotune
  Info: Access autotune tunables using: kubectl -n monitoring get autotuneconfig
  ```

#### What does it not do?
  It does not kick off any experiments with the tunables (as yet). Stay tuned!!

#### **pre-req**
  It expects minikube to be installed with atleast 8 CPUs and 16384MB Memory.

##### WARNING: The script deletes any existing minikube cluster.

### What is an Objective Function?

An objective function specifies a tuning goal in the form of a monitoring system (Eg Prometheus) query.
```
apiVersion: "recommender.com/v1"
kind: Autotune
metadata:
  name: "galaxies-autotune"
  namespace: "default"
spec:
  slo:
    objective_function: "request_sum/request_count"
    slo_class: "response_time"
    direction: "minimize"
    hpo_algo_impl: "optuna_tpe_multivariate"
    function_variables:
    - name: "request_sum"
      query: rate(http_server_requests_seconds_sum{method="GET",outcome="SUCCESS",status="200",uri="/galaxies",}[1m])
      datasource: "prometheus"
      value_type: "double"
    - name: "request_count"
      query: rate(http_server_requests_seconds_count{method="GET",outcome="SUCCESS",status="200",uri="/galaxies",}[1m])
      datasource: "prometheus"
      value_type: "double"
  mode: "show"
  selector:
    matchLabel: "app.kubernetes.io/name"
    matchLabelValue: "galaxies-deployment"
    matchRoute: ""
    matchURI: ""
    matchService: ""
```
In the above yaml from the [benchmarks](https://github.com/kruize/benchmarks/blob/master/galaxies/autotune/autotune-http_resp_time.yaml) repo, the overall goal or `slo` for the IT Admin is to minimize response time of the application deployment `galaxies-deployment`. The objective function defines what exactly constitutes response time. In this example it is a regular expression `request_sum/request_count`. The `function_variables` section helps to resolve the individual variables of the expression. In this case, `request_sum` represents the value returned by the prometheus query `rate(http_server_requests_seconds_sum{method="GET",outcome="SUCCESS",status="200",uri="/galaxies",}[1m])` and `request_count` represents the value returned by the prometheus query `rate(http_server_requests_seconds_count{method="GET",outcome="SUCCESS",status="200",uri="/galaxies",}[1m])`. Autotune uses the `optuna_tpe` hyper parameter optimization algorithm by default. However you can use `hpo_algo_impl` to change it to use a different optuna supported HPO algorithm.

See the [benchmarks](https://github.com/kruize/benchmarks) repo for more examples on how to define Autotune objects. Look for ${benchmark_name}/autotune dir for example yamls.

### What is a Layer?

A Layer is a software component of an application stack. In Autotune, we currently have 4 layers defined.
```
layer 0 = Container (Fixed and is always present)
layer 1 = Language Runtime (Eg. Hotspot JVM, Eclipse OpenJ9 JVM, nodejs...)
layer 2 = App Server / Framework (Quarkus, Liberty, Springboot...)
layer 3 = Application
```
A layer is defined by an `AutotuneConfig` object in Autotune. See the layer [template](https://github.com/kruize/autotune/blob/master/manifests/autotune-configs/layer-config.yaml_template) if you want to add a new layer.

### What is a Tunable?

A tunable is a performance tuning knob specific to a layer. Autotune currently supports `double` and `integer` tunables. Tunables of these two types require a range to be specified in the form of a `upper_bound` and a `lower_bound`. This forms the valid range of values inside which the tunable operates. Tunables are defined in the `AutotuneConfig` object and are associated with a layer. Eg. The [Quarkus](https://github.com/kruize/autotune/blob/master/manifests/autotune-configs/quarkus-micrometer-config.yaml) `AutotuneConfig` yaml currently defines three tunables. See the layer [template](https://github.com/kruize/autotune/blob/master/manifests/autotune-configs/layer-config.yaml_template) if you want to add a new tunable.

### What is `slo_class`?

`slo_class` is associated with a tunable and is one or more of `response_time`, `throughput` and `resource_usage`. It represents the impact a tunable has on the outcome for any given workload. For example, we know based on prior experiments that the [hotspot](https://github.com/kruize/autotune/blob/master/manifests/autotune-configs/hotspot-micrometer-config.yaml) tunable, `MaxInlineLevel` has an impact on both `throughput` and `response_time` but has a negligible impact on `resource_usage`.

### How do I run it?

```
# To setup the demo. This will use the default docker images as decided
# by the autotune version in pom.xml in the autotune repo
$ ./autotune_minikube_demo_setup.sh

# If you want to access the Prometheus Console
$ ./autotune_minikube_demo_setup.sh -sp

# If you want to restart only autotune with the specified docker image
$ ./autotune_minikube_demo_setup.sh -r -i [autotune operator image] -o [autotune optuna image]

# To terminate the demo
$ ./autotune_minikube_demo_setup.sh -t
```

## Info about Kruize repositories

- [kruize-demos](https://github.com/kruize/kruize-demos)
  This repo.
- [autotune](https://github.com/kruize/autotune)
  Main repo for Autotune sources.
- [benchmarks](https://github.com/kruize/benchmarks)
  Benchmark scripts for running performance tests with Autotune.
- [autotune-results](https://github.com/kruize/autotune-results)
  Results of experiments run with Autotune.
- [hpo](https://github.com/kruize/hpo)
  HPO as a Service.
