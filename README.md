# Knative Kafka Operator

The following will install Knative Kafka and configure it
appropriately for your cluster in the `default` namespace:

    kubectl apply -f deploy/crds/eventing_v1alpha1_knativeeventingkafka_crd.yaml
    kubectl apply -f deploy/
    kubectl apply -f deploy/crds/eventing_v1alpha1_knativeeventingkafka_cr.yaml

## Prerequisites

### Apache Kafka

This operator installs components that access Apache Kafka, therefore you MUST have a running
cluster of Apache Kafka _somewhere_.

   - For Kubernetes a simple installation is done using the
     [Strimzi Kafka Operator](http://strimzi.io). Its installation
     [guides](http://strimzi.io/quickstarts/) provide content for Kubernetes and
     Openshift.

### Operator SDK

This operator was created using the
[operator-sdk](https://github.com/operator-framework/operator-sdk/).
It's not strictly required but does provide some handy tooling.

## The KnativeEventingKafka Custom Resource

The installation of Knative Kafka is triggered by the creation of
[an `KnativeEventingKafka` custom
resource](deploy/crds/eventing_v1alpha1_knativeeventingkafka_crd.yaml).

The following are all equivalent, but the latter may suffer from name
conflicts.

    kubectl get knativeventingkafka.eventing.knative.dev -oyaml
    kubectl get kek -oyaml
    kubectl get knativeventingkafka -oyaml

To uninstall Knative Kafka, simply delete the `KnativeEventingKafka` resource.

    kubectl delete kek --all

## Development

It can be convenient to run the operator outside of the cluster to
test changes. The following command will build the operator and use
your current "kube config" to connect to the cluster:

    operator-sdk up local --namespace=""

Pass `--help` for further details on the various `operator-sdk`
subcommands, and pass `--help` to the operator itself to see its
available options:

    operator-sdk up local --operator-flags "--help"

### Building the Operator Image

To build the operator,

    operator-sdk build quay.io/$REPO/knative-kafka-operator:$VERSION

The image should match what's in
[deploy/operator.yaml](deploy/operator.yaml) and the `$VERSION` should
match [version.go](version/version.go) and correspond to the contents
of [deploy/resources](deploy/resources/).

There is a handy script that will build and push an image to
[quay.io](https://quay.io/repository/openshift-knative/knative-kafka-operator)
and tag the source:

    ./hack/release.sh
	
## Operator Framework

The remaining sections only apply if you wish to create the metadata
required by the [Operator Lifecycle
Manager](https://github.com/operator-framework/operator-lifecycle-manager)

### Create a ClusterServiceVersion

The OLM requires special manifests that the operator-sdk can help
generate.

Create a `ClusterServiceVersion` for the version that corresponds to
the manifest[s] beneath [deploy/resources](deploy/resources/). The
`$PREVIOUS_VERSION` is the CSV yours will replace.

    operator-sdk olm-catalog gen-csv \
        --csv-version $VERSION \
        --from-version $PREVIOUS_VERSION \
        --update-crds

Most values should carry over, but if you're starting from scratch,
some post-editing of the file it generates may be required:

* Add fields to address any warnings it reports
* Verify `description` and `displayName` fields for all owned CRD's

### Create a CatalogSource

The [catalog.sh](hack/catalog.sh) script should yield a valid
`ConfigMap` and `CatalogSource` comprised of the
`ClusterServiceVersions`, `CustomResourceDefinitions`, and package
manifest in the bundle beneath
[deploy/olm-catalog](deploy/olm-catalog/). You should apply its output
in the namespace where the other `CatalogSources` live on your cluster,
e.g. `openshift-marketplace`:

    CN_NS=$(kubectl get catalogsources --all-namespaces | tail -1 | awk '{print $1}')
    ./hack/catalog.sh | kubectl apply -n $CN_NS -f -

### Using OLM on Minikube

You can test the operator using
[minikube](https://kubernetes.io/docs/setup/minikube/) after
installing OLM on it:

    minikube start
    kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.9.0/olm.yaml

Once all the pods in the `olm` namespace are running, install the
operator like so:
    
    ./hack/catalog.sh | kubectl apply -n $CN_NS -f -

Interacting with OLM is possible using `kubectl` but the OKD console
is "friendlier". If you have docker installed, use [this
script](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/scripts/run_console_local.sh)
to fire it up on <http://localhost:9000>.

#### Using kubectl

To install Knative Kafka into the `knative-eventing` namespace, apply
the following resources:

```
CN_NS=$(kubectl get catalogsources --all-namespaces | tail -1 | awk '{print $1}')
OPERATOR_NS=$(kubectl get og --all-namespaces | grep global-operators | awk '{print $1}')
cat <<-EOF | kubectl apply -f -
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: knative-kafka-operator-sub
  generateName: knative-kafka-operator-
  namespace: $OPERATOR_NS
spec:
  source: knative-kafka-operator
  sourceNamespace: $CN_NS
  name: knative-kafka-operator
  channel: alpha
---
apiVersion: eventing.knative.dev/v1alpha1
kind: KnativeEventingKafka
metadata:
  name: knative-eventing-kafka
  namespace: knative-eventing
spec:
  bootstrapServers: my-cluster-kafka-bootstrap.kafka:9092
  #setAsDefaultChannelProvisioner: yes
EOF
```

### Releasing new version

Make sure you updated `version/version.go` first. 
Also update `deploy/operator.yaml` with the new image.

Replace the old files in `deploy/resources` with the new ones.
(e.g. create `deploy/resources/kafka-channel-v0.12.1.yaml` with https://github.com/knative/eventing-contrib/releases/download/v0.12.1/kafka-channel.yaml) 

Then run these commands to generate OLM metadata for the new version of the operator:

```
DIR=${DIR:-$(pwd)}
NAME=${NAME:-$(ls $DIR/deploy/olm-catalog)}

# find the latest version from nested directories in deploy/olm-catalog/knative-eventing-operator
LATEST_VERSION=$(find deploy/olm-catalog/${NAME}/* -maxdepth 1 -type d -exec basename {} \; | sort -V | tail -1)

# read the current/next version from version/version.go
CURRENT_VERSION=$(awk '/Version =/{print $NF}' version/version.go | awk '{gsub(/"/, "", $1); print $1}')
                                
# if you have operator-sdk CLI version < 0.15.0
operator-sdk olm-catalog gen-csv --update-crds --csv-version "${CURRENT_VERSION}" --from-version "${LATEST_VERSION}"

# if you have operator-sdk CLI version >= 0.15.0
# GO111MODULE="on" operator-sdk generate csv --update-crds --csv-version "${CURRENT_VERSION}" --from-version "${LATEST_VERSION}"
```

Then look for references to previous version in the `deploy/olm-catalog/knative-kafka-operator/${CURRENT_VERSION}/*.clusterserviceversion.yaml` file.
Replace them with the new version. ...except the `replaces` property.

Add following to the `deploy/olm-catalog/knative-kafka-operator/${CURRENT_VERSION}/*.clusterserviceversion.yaml` file. 

```
args:
  - --filename=https://raw.githubusercontent.com/openshift/knative-eventing-contrib/release-v${CURRENT_VERSION}/openshift/release/knative-eventing-kafka-contrib-v${CURRENT_VERSION}.yaml,https://raw.githubusercontent.com/openshift-knative/knative-kafka-operator/v${CURRENT_VERSION}/deploy/resources/networkpolicies.yaml
```

The `networkpolicies.yaml` file won't be there yet. However, with the release cut, it will be available.
