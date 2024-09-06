# Helm Notes
Sources:
- [Helm docs](https://helm.sh/docs/)

Helm is the package manager for Kubernetes. It lets you define, install, and upgrade your Kubernetes application.

## Core Concepts
Helm installs charts into Kubernetes, creating a new release for each installation. To find new charts, you can search Helm chart repositories. 

A **Chart** is a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside a Kubernetes cluster. You can think of it like the Kubernetes equivalent of a Homebrew formula.

A **Repository** is the place where charts can be collected and shared. It's like the Fedora Package Database, but for Kubernetes packages.

A **Release** is an instance of a chart running in a Kubernetes cluster. One chart can often be installed many times in the same cluster. Each time it's installed, a new release is created. For example, if you want two MySQL databases running in your cluster, you can install a MySQL chart twice. Each one will have its own release, which wil in turn have its own release name.

## Helm Architecture
Helm is a tool for managing packages called charts. It can do the following:
- Create new charts from scratch
- Package charts into chart archive (tgz) files
- Interact with chart repositories where charts are stored
- Install charts into and uninstall charts from an existing Kubernetes cluster
- Manage the release cycle of charts that have been installed with Helm

For Helm, there are three important concepts:
1. The chart is a bundle of information necessary to create an instance of a Kubernetes application
2. The config contains configuration information that can be merged into a packaged chart to create a releasable object
3. A release is a running instance of a chart, combined with a specific config.

### Components
Helm is an executable which is implemented in two distinct parts

The **Helm Client** is a command-line client for end users. The client is responsible for the following:
- Local chart development
- Managing repositories
- Managing releases
- Interfacing with the Helm library
	- Sending charts to be installed
	- Requesting upgrading or uninstalling of existing releases

The **Helm Library** provides the logic for executing all Helm operations. It interfaces with the Kubernetes API server and provides the following capability:
- Combining a chart and configuration to build a release
- Installing charts into Kubernetes, and providing the subsequent release object
- Upgrading and uninstalling charts by interacting with Kubernetes

The standalone Helm library encapsulates the Helm logic so that it can be leveraged by different clients.

### Implementation
The Helm Client and Library are written in Go.

The Library uses the Kubernetes cient library to communicate with Kubernetes. Currently that library uses REST + JSON. It stores information in Secrets located inside of Kubernetes and doesn't need its own database.

Configuration files are, when possible, written in YAML.

## Charts
Helm uses a packaging format called charts. A chart is a collection of files that describe a related set of Kubernetes resources. A single chart might be used to deploy simething simple, like a memcached pod, or something complex, like a fullstack web app with HTTP servers, databases, caches, and so on.

Charts are created as files laid out in a particular directory tree. They can be packaged into versioned archives to be deployed.

If you want to download and look at the files for a published chart, without installing it, you can use `helm pull chartrepo/chartname`.

### The Chart File Structure
A chart is organized as a collection of files inside a directory. The directory name is the name of the chart (without versioning information), so a chart describing WordPress would be stored in a `wordpress/` directory.

Inside this directy, Helm will expect the following structure:

```
wordpress/
	Chart.yaml
	LICENCE              # OPTIONAL 
	README.md            # OPTIONAL 
	values.yaml 
	values.schema.json   # OPTIONAL 
	charts/ 
	crds/
	templates/
	templates/NOTES.txt  # OPTIONAL 
```

- `Chart.yaml` - A YAML file containing information about the chart
- `LICENCE` (optional) - A plaintext file containing the licence for the chart
- `README.md` (optional) - A human-readable README file
- `values.yaml` - The default configuration values for this chart
- `values.schema.json` (optional) - A JSON schema for imposing a structure on values.yaml
- `charts/` - A directory containing any charts this chart depends on
- `crds/` - Custom resource definitions
- `templates/` - A directory of templates that, when combined with values, will generate valid Kubernetes manifest files
- `templates/NOTES.txt` - A plaintext file containing short usage notes

Helm reserves the use of the `charts/`, `crds/`, and `templates/` directories, and of the listed file names. Other files wil be left as they are.

### The `Chart.yaml` File

The `Chart.yaml` file is required for a chart. It contains the following fields:

```yaml
apiVersion:
name:
version:
kubeVersion:   # OPTIONAL
description:   # OPTIONAL
type:          # OPTIONAL
keywords:      # OPTIONAL
  -
home:          # OPTIONAL
sources:       # OPTIONAL
  - 
dependencies:  # OPTIONAL
  - 
maintainers:   # OPTIONAL
  - 
icon:          # OPTIONAL
appVersion:
deprecated:    # OPTIONAL
annotations:   # OPTIONAL
  - 
```

- `apiVersion` - The chart API version
- `name` - The name of the chart
- `version` - A SemVer 2 version
- `kubeVersion` (optional) - A SemVer range of compatible Kubernetes versions
- `description` (optional) - A single sentence description of the project.
- `type` (optional) - The type of the chart
- `keywords` (optional) - A list of keywords about the project
- `home` (optional) - The URL of this project's homepage
- `sources` (optional) - A list of URLs to source code for this project
- `dependencies` (optional) - A list of the chart requirements
	- `name` - The name of the chart, such as nginx
	- `version` - The version of the chart, such as "1.2.3"
	- `repository` (optional) - The repository URL ("https://example.com/charts") or alias ("@repo-name")
	- `condition` (optional) - A YAML path that resolves to a boolean, used for enabling/disabling charts
	- `tags` (optional) - Tags can be used to group charts for enabling/disabling
	- `import-values` (optional) - Holds the mapping of source values to parent key to be imported. Each item can be a string or pair of child/parent sublist items
	- `alias` (optional) - Alias to be used for the chart. Useful when you have to add the same chart multiple times
- `maintainers` (optional):
	- `name` - The maintainer's name
	- `email` (optional): The maintainer's email
	- `url` (optional): A URL for the maintainer
- `icon` (optional) - A URL to an svg/png image to be used as an icon
- `appVersion` (optional) - The version of the app that this contains. Needn't be SemVer. Quotes recommended.
- `deprecated` (optional) - Whether the chart is deprecated (boolean)
- `annotations` (optional) - A list of annotations keyed by name

From version 3.3.2, additional fields are not allowed. The recommended approach is to add custom metadata in `annotations`.

#### Charts and Versioning
Every chart must have a version number, which must follow the SemVer 2 standard. Helm 2+ uses version numbers as release markers. Packages in repositories are identified by name plus version.

For example, an `nginx` chart whose version field is set to `version: 1.2.3` will be named: `nginx-1.2.3.tgz`.

More complex SemVer 2 names are also supported, such as `version: 1.2.3-alpha.1+ef365`. But non-SemVer names are explicitly disallowed by the system.

The `version` field in `Chart.yaml` is used by many Helm tools, including the CLI. When generating a package, the `helm package` command will use the version that it finds in `Chart.yaml` as a token in the package name. The system assumes that the version number in the chart package name matches the version number in `Chart.yaml`. Failure to meet this assumption will cause an error.

#### The `apiVersion` Field
The `apiVersion` field should be `v2` for Helm charts that require at least Helm 3. Charts supporting previous Helm versions have an `apiVersion` set to `v1` and are still installable by Helm 3.

Changes from `v1` to `v2`:
- A `dependencies` field defining chart dependencies, which were located in a seperate `requirements.yaml` file for `v1` charts
- The `type` field, distinguishing between application and library charts

#### The `appVersion` Field
The `appVersion` field specifies the version of the application, and is not related to the `version` field.

For example, a Drupal chart may have `appVersion: "8.2.1"`, indicating that the version of Drupal included in the chart by default is 8.2.1. Wrapping the version in quotes is highly recomended as it forces the YAML parser to treat the version number as a string. 

As of Helm 3.5, `helm create` wraps the default `appVersion` field in quotes.

#### The `kubeVersion` Field
The optional `kubeVersion` field can define SemVer constraints on supported Kubernetes versions. Helm will validate the version constraints when installing the chart and will fail if the cluster runs an unsupported Kubernetes version.

AND comparisons are space-seperated, such `>= 1.13.0 < 1.15.0`

These can be combined with the OR operator

```
>= 1.13.0 < 1.14.0 || >= 1.14.1 < 1.15.0
```

Hyphen ranges are used for closed intervals, so `1.1 - 2.3.4` is equivalent to `>= 1.1 <= 2.3.4`.

Wildcards are `x`, `X` and `*`, so `1.2.x` is equivalent to `>= 1.2.0 < 1.3.0`.

Tilde ranges allow patch version changes, so `~1.2.3` is equivalent to `>= 1.2.3 < 1.3.0`.

Caret ranges allow minor version changes, so `^1.2.3` is equivalent to `>= 1.2.3 < 2.0.0`.

#### Deprecating Charts
When managing charts in a Chart Repository, it's sometimes necessary to deprecate a chart. The optional `deprecated` field in `Chart.yaml` can be used to mark a chart as deprecated. 

If the latest version of a chart in the repository is deprecated, then the chart as a whole is considered deprecated. The chart name can be later reused by publishing a newer version that is not marked as deprecated. 

The workflow for deprecating charts is:
1. Update the chart's `Chart.yaml` to mark the chart as deprecated, bumping the version
2. Release the new chart version in the Chart Repository
3. Remove the chart from the source repository (such as Git)

#### Chart Types
The `type` field defines the type of the chart: either `application` or `library`. Application is the default type and is the standard chart which can be operated on fully. The library chart provides utilities or functions for the chart builder. A library chart differs from an application chart because it is not installable and usually doesn't contain any resource objects.

Note: An aplication chart can be used as a library chart. This is enabled by setting the type to `library`. The chart will then be rendered as a library chart where all utilities and functions can be rendered. All resource objects of the chart will not be rendered.

### Chart LICENCE, README and NOTES
Charts can also contain files that describe the installation, configuration, usage and licence of a chart.

A LICENCE is a plaintext file containing the licence for the chart. There can also be seperate licences for the application installed by the chart, if necessary.

A README for a chart should be formatted in Markdown and should generally contain:
- A description of the application or service the chart provides
- Any prerequisites or requirements to run the chart
- Descriptions of options in `values.yaml` and default values
- Any other information that may be relevant to the installation or configuration of the chart

When hubs and other UIs display details about a chart, that detail is pulled from the content in `README.md`.

The chart can also contain a short plaintext `templates/NOTES.txt` file that will be printed out after installation, and when viewing the status of a release. It's recommended to keep the content brief and point to the README for greater detail.

### Chart Dependencies
In Helm, one chart may depend on any number of other charts. These dependencies can be dynamically linked using the `dependencies` field in `Chart.yaml` or brought into the `charts/` directory and managed manually.

#### Managing Dependencies with the `dependencies` field
The charts required by the current chart are defined as a list in the `dependencies` field.

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: https://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: https://another.example.com/charts
```

- name - the name of the chart you want
- version - the version of the chart you want
- repository - the full URL to the chart repository. You must also use `helm repo add` to add the repo locally

You might use the name of the repo instead of a URL

```bash
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
```

```yaml
dependencies:
  - name: awesomeness
    version: 1.0.0
    repository: "@fantastic-charts"
```

Once you have defined dependencies, you can run `helm dependency update` and it will use your dependency file to download all the specified charts into your `charts/` directory for you.

When `helm dependency update` retrieves charts, it will store them as chart archives in the `charts/` directory, so:

```
charts/
	apache-1.2.3.tgz
	mysql-3.2.1.tgz
```

#### `alias` Field in Dependencies
Adding an alias for a dependency chart would put a chart in dependencies using the alias as the name of the new dependency.

You can use `alias` in cases where you need to access a chart with other names.

#### Tags and Condition Fields in Dependencies   
All charts are loaded by default. If the `tags` or `condition` fields are present, they will be evaluated and used to control loading for the charts they are applied to.

The `condition` field holds one or more YAML paths, delimited by commas. If this path exists in the top parent's values and resolves to a boolean, the chart will be enabled or disabled based on that boolean value. Only the first valid path  found in the list is evaluated and if no paths exist then the condition has no effect.

The `tags` field is a YAML list of labels to associate with this chart. In the top parent's values, all charts with tags can be enabled or disabled by specifying the tag and a boolean value.

#### Using the CLI with Tags and Conditions
The `--set` parameter can be used to alter tag and condition values.

```bash
helm install --set tags.front-end=true --set subchart2.enabled=false
```

#### Tags and Condition Resolution
Conditions, when set in values, always override tags. The first condition path that exists wins and subsequent ones for that chart are ignored.

Tags are evaluated where if any of the chart's tags are true then the chart is enabled.

Tags and condition values must be set in the top parent's values.

The `tags` key must be a top-level key. Globals and nested `tags` are not currently supported.

#### Importing Child Values Via Dependencies
In some cases it's desireable to allow a child chart's values to propagate to the parent chart and be shared as common defaults. An additional benefit of using the `exports` format is that it will enable future tooling to introspect user-settable values.

The keys containing the values to be imported can be specified in the parent chart's `dependencies` in the field `import-values` using a YAML list. Each item in the list is a key which is imported from the child chart's `exports` field.

#### Managing Dependencies Manually via the `charts/` Directory
Dependencies can be expressed explicitly by copying the dependency charts into the `charts/` directory. A dependency should be an unpacked chart directory. It's name cannot start with `_` or `.`, such filenames are ignored by the chart loader. 

For example, if a WordPress chart depends on an Apachew chart, the correct version of the Apache chart should be supplied in the WordPress chart's `charts/` directory.

To drop a dependency into your `charts/` direcotry, use the `helm pull` command.

```
wordpress:
	Charts.yaml
	# ...
	charts/
		apache/
			Chart.yaml
			# ...
```

#### Operational Aspects of Using Dependencies
How do chart dependencies affect chart installation using `helm install` and `helm upgrade`?

When Helm installs/upgrades charts, the Kubernetes objects from the charts and all its dependencies are
- aggregated into a single set
- sorted by type, followed by name
- created/updated in that order

This creates a single release with all the objects for the chart and its dependencies. 

### Templates and Values
Helm chart templates are written in the Go template language, with additional functons from the Sprig library and a few other specialized functions.

All template files are stored in a chart's `templates/` folder. When Helm renders the charts, it will pass every file in that directory through the template engine.

Values for templates are supplied two ways:
- Chart developers may supply a `values.yaml` file inside a chart. This file can contain default values.
- Chart users may supply a YAML file that cntains values. This can be provided on the command line with `helm install`.

When a user supplies custom values, these values will override the values in the chart's `values.yaml` file.

#### Template Files
Template files follow the standard conventions for writing Go templates.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata: 
	  labels:
	    app.kubernetes.io/name: deis-database
	spec:
	  serviceAccount: deis-database
	  containers:
	    - name: deis-database
		  image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
		  imagePullPolicy: {{ .Values.pullPolicy }}
		  ports:
		    - containerPort: 5432
		  env:
		    - name: DATABASE_STORAGE
			  value: {{ default "minio" .Values.storage }}
```

In the example above, the following template values are being injected from a `values.yaml` file:
- `imageRegistry` - The source registry for the Docker image
- `dockerTag` - The tag for the Docker image
- `pullPolicy` - The Kubernetes pull policy
- `storage` - The storage backend, whose default is set to "minio"

All of these values are defined by the template author. Helm does not require or add parameters.

#### Predefined Values
Values that are supplied via a `values.yaml` file, or va the `--set` flag, are accessible from the `.Values` object in a template. But there are other pre-defined pieces of data you can access in your templates.

The following values are pre-defined, accessible t every template, and cannot be overridden. As with all values, the names are case sensitive.

- `Release.Name` - The name of the release (not the chart)
- `Release.Namespace` - The namespace the chart was released to
- `Release.Service` - The service that conducted the release
- `Release.IsUpgrade` - This is set to true if the current operation is an upgrade or rollback
- `Release.IsInstall` - This is set to true if the current operation an install
- `Chart` - The contents of `Chart.yaml`, so the chart version is accessible as `Chart.Version` and the maintainers as `Chart.Maintainers`
- `Files` - A map-like object containing all non-special files in the chart. This will not give you access to templates but will give you access to additional files that are present (unless you excluded them using `.helmignore`). Files can be accessed using `{{ index.Files "file.name" }}` or using the `{{ .Files.Get name }}` function. You can also access the contents of the file as `[]byte` using `{{ .Files.GetBytes }}`
- `Capabilities` - A map-like object that cntains information about the versions of Kubernetes `{{ .Capabilities.KubeVersion }}` and the supported Kubernetes API versions `{{ .Capabilities.APIVersions.Has "batch/v1"}}`

Any unknown `Chart.yaml` fields will be dropped and will not be accessible inside the `Chart` object. `Chart.yaml` cannot be used to pass arbitrarly structured data into the template. The values file can be used for that, though.

#### Values Files
The default values file included inside a chart must be named `values.yaml`. Files specified on the command line can be named anything.

A values file is formatted in YAML. A chart may include a default `values.yaml` file. The `helm install` command allows a user to override values by supplying additional YAML values.

```bash
helm install --generate-name --values=myvals.yaml wordpress
```

When values are passed in this way, they will be merged into the default values file. For example, if a `values.yaml` file contains:

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

And a `myvals.yaml` file containing `storage: "gcs"`  is merged with it, the `storage` field will be overridden.

Any of these values are then accessible inside of templates by using the `.Values` object.

If the `--set` flag is used on `helm install` or `helm upgrade`, those values are simply converted to YAML n the client side.

If any required entries in the values file exist, they can be declared as required in the chart template by using the `required` function.

#### Scope, Dependencies, and Values
A values file can supply values to the chart as well as to any of its dependencies.

A values file for a WordPress chart with MySQL and Apache dependencies culd supply values to all of these cmponents.

```yaml
title: "My WordPress site"

mysql:
  max_connections: 100
  password: "secret"

apache: 
  port: 8080
```

Charts at a higher level have access to all the variables defined beneath. So the WordPress chart can access the MySQL password via `.Values.mysql.password`. But lower charts cannot access things in the parent charts, so MySQL will not be able to access the `title` prperty, or `apache.port`.

Values are namespaced, but namespaces are pruned. So the WordPress chart can access the MySQL password via `.Values.mysql.password`. But for the MySQL chart, the scope of the values has been reduced and the namespace prefix removed, so it will use `.Values.password`.

#### Global Values
Helm 2+ supports "global" values.

```yaml
title: "My WordpPress site"

global:
  app: mMyWordPress
```

This value is available to **all** charts as `.Values.global.app`.

This provides a way of sharing one top-level variable with all subcharts, which is useful for things like setting `metadata` properties like labels.

If a subchart declares a global variable, that global will be passed downward to the subchart's subcharts but not upward to the parent chart. There is no way for a subchart to influence the values of the parent chart. 

Global variables of parent charts take precedence over the global variables from subcharts.

#### Schema Files
Sometimes, a chart maintainer might want to define a srtructure on their values. This can be done by defining a schema in the `values.schema.json` file. A schema is represented as a JSON schema, such as:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "image": {
      "description": "Container Image",
      "properties": {
        "repo": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      },
      "type": "object"
    },
    "name": {
      "description": "Service name",
      "type": "string"
    },
    "port": {
      "description": "Port",
      "minimum": 0,
      "type": "integer"
    },
    "protocol": {
      "type": "string"
    }
  },
  "required": [
    "protocol",
    "port"
  ],
  "title": "Values",
  "type": "object"
}
```

This schema will be applied to the values to validate it. Validation occurs when any of the following commands are invoked:
- `helm install`
- `helm upgrade`
- `helm lint`
- `helm template`

The schema is applied to the final `.Values` object and not just to the `values.yaml` file. The final `.Values` object is checked against all subchart schemas. This means that restrictions on a subchart can't be circumvented by a parent chart. Conversely, if a subchart has a requirement that is not met in the subchart's `values.yaml` file, the parent chart **must** satisfy those restrictions in order to be valid.

### Custom Resource Definitions (CRDs)
Kubernetes provides a mechanism for declaring new types of Kubernetes objects. Using CustomResourceDefinitions (CRDs), Kubernetes developers can declare custom resource types.

In Helm 3, CRDs are treated as a special kind of object. They are installed before the rest of the chart, and are subject to some limitations.

CRD YAML files should be placed in the `crds/` directory inside of a chart. Multiple CRDs (seperated by YAML start and end markers) may be placed in the same file.Helm will attempt to load all of the files in the CRD directory into Kubernetes.

CRD files cannot be templated. They must be plain YAML documents.

When Helm installs a new chart, it will upload the CRDs, pause until the CRDs are made available by the API server, then start the template engine, render the rest of the chart, and upload it to Kubernetes. Because of this ordering, CRD information is available in the `.Capabilities` object in Helm templates, and Helm templates may create new instances of objects that were decalred in CRDs.

For example, if your chart had a CRD for `CronTab` in the `crds/` directory, you can create instances of the `CronTab` kind in the `templates/` directory:

```
crontabs/
	Chart.yaml
	crds/
		crontab.yaml
	templates/
		mycrontab.yaml
```

The `crontab.yaml` file must contain the CRD with no template directives

```yaml
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
```

Then the template `mycrontab.yaml` can create a new `CronTab` using templates as usual:

```yaml
apiVersion: stable.example.com
kind: CronTab
metadata:
  name: {{ .Values.name }}
spec:
   # ...
```

Helm will make sure that the `CronTab` kind has been installed and is available from the Kubernetes API server before it proceeds installing the things in `templates/`.

#### Limitations on CRDs
Unlike most objects in Kubernetes, CRDs are installed globally. For that reason, Helm takes a very cautious approach in managing CRDs. They are subject to the following limitations:
- CRDs are never reinstalled. If Helm determines that the CRDs in the `crds/` directory are already present (regardless of version), Helm will not attempt to install or upgrade.
- CRDs are never installed or upgraded on rollback. Helm will only create CRDs on installation operations.
- CRDs are never deleted. Deleting a CRD would automatically delete all of the CRD's contents across all namespaces in the cluster.

Operators who want to upgrade or delete CRDs are encouraged to do this manually and with great care.

### Using Helm to Manage Charts
The `helm` tool has several commands for working with charts.

You can create a new chart:

```bash
helm create mychart
```

Once you have edited a chart, `helm` can package it into a chart archive for you:

```bash
helm package mychart
```

You can also use `helm` to help you find issues with your chart's formatting or information:

```bash
helm lint mychart
```

### Chart Repositories
A chart repository is a HTTP server that houses one or more packaged charts. While `helm` can be used to manage local chart directories, the preferred mechanism for sharing charts is a chart repository.

Any HTTP server that can server YAML and tar files and can answer GET requests can be used as a repository server (this includes Google Cloud Storage and S3 with website mode enabled.

A repository is characterized primarily by the presence of an `index.yaml` file that has a list of all the packages supplied by the repository, together with metadata that allows retrieving and verifying those packages.

On the client side, repositories are managed with the `helm repo` commands. However, Helm does not provide tools for uploading charts to remote repository servers.

### Chart Starter Packs
The `helm create` command takes an optional `--starter` option that lets you specify a "starter chart".

Starters are just regular charts but are located in `$XDG_DATA_HOME/helm/starters`. A chart developer can author charts that are specifcally designed to be used as starters. These charts should be designed with the following in mind:
- The `Chart.yaml` will be overwritten by the generator
- Users wil expect to modify such a chart's contents, so documentation should indicate how to do so
- All occurances of `<CHARTNAME>` will be replaced with the specified chart name so that the starter can be used as a template

Currently, the only way to add a chart to `$XDG_DATA_HOME/helm/starters` is to manually copy it there. This process should be explained in the chart's documentation.

## Chart Hooks
Helm provides a hook mechanism to allow chart developers to intervene at certain points in a release's lifecycle. For example, you can use hooks to:
- Load a ConfigMap or Secret during install before any other charts are loaded
- Execute a Job to back up a database before installing a new chart, and then execute a second job after the upgrade in order to restore data
- Run a Job before deleting a release to gracefully take a service out of rotation before removing it

Hooks work like regular templates but have special annotations that cause Helm to use them differently.

#### Available Hooks
- `pre-install` - Executes after templates are rendered, but before any resources are created in Kubernetes
- `post-install` - Executes after all resources are loaded into Kubernetes
- `pre-delete` - Executes on a deletion request before any resources aare deleted from Kubernetes
- `post-delete` - Executes on a deletion request after all of the release's resources have been deleted
- `pre-upgrade` - Executes on an upgrade request after templates are rendered, but before any resources are updated
- `post-upgrade` - Executes on an upgrade request after all resources have been upgraded
- `pre-rollback` - Executes on a rollback request after templates are rendered, but before any resources are rolled back
- `post-rollback` - Executes on a rollback request after all resources have been modified
- `test` - Executes when the Helm test subcommand is invoked

#### Hooks and the Release Lifecycle
Hooks allow the chart developer to perform operations at strategic points in a release lifecycle.

What does it mean to wait until a hook is ready? This depends on the resource declared in the hook. If the resource is a `Job` or Pod `kind`, Helm will wait until it successfully runs to completion. And if the hook fails, the release will fail. This is a blocking operation, so the Helm client will pause while the Job is run.

For all other kinds, as soon as Kubernetes marks the resource as loaded (added or updated), the resource is considered "ready". When many resources are declared in a hook, the resources are executed serially. If they have hook weights, they are executed in weighted order. It's a good practice to set a hook weight and set it to 0 if weight is not important.

If you create resources in a hook, you cannot rely on `helm uninstall` to remove the resources. To destroy these resources, you need to either add a custom `helm.sh/hook-delete-policy` to the hook template file, or set the time to live field of a Job resource.

### Writing a Hook
Hooks are just Kubernetes manifest files with special annotations in the `metadata` section. Because they are template files, you can use all the normal template features, including reading `.Values`, `.Release`, and `.Template`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}"
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
    app.kubernetes.io/instance: {{ .Release.Name | quote }}
    app.kubernetes.io/version: {{ .CHart.AppVersion }}
    helm.sh/chart: "{{ .Chart.Name }}-{{ .CHart.Version }}"
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
# ...
```

What makes this template a hook is the annotation:

```yaml
annotations:
  "helm.sh/hook": post-install
```

One resource can implement multiple hooks:

```yaml
annotations:
  "helm.sh/hook": post-install, post-upgrade
```

There is no limit to the number of different resources that may implement a given hook. For example, you could declare both a Secret and a ConfigMap as a pre-install hook.

When a subchart declares hooks, those are also evaluated. There is no way for a top-level chart to disable the hooks declared by subcharts.

Hooks can be assigned weights to build a deterministic execution order. They can be positive or negative number but must be represented as strings. When Helm starts the execution cycle of hooks of a particular kind it will sort those hooks in ascending order.

#### Hook Deletion Policies
It's possible to define policies that determine when to delete corresponding hook resources.

```yaml
annotations:
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

- `before-hook-creation` - Delete the previous resource before a new hook is launched (default if no policy specified)
- `hook-succeeded` - Delete the resource after the hook is successfully executed
- `hook-failed` - Delete the resource if the hook failed during execution

## Using Helm
`helm help` will provide nfrmation on available commands.

### `helm search` - Finding Charts
Helm can search two different types of sources.
- `helm search hub` searches the Artifact Hub, which lists Helm charts from dozens of different repositories
- `helm search repo` searches the repositories that you have aded to your local helm client (with `helm repo add`). This search is done over local data and no public network connection is needed.

You can find publicly available charts by running `helm search hub`. With no filter, it will show you all of the available charts.

Using `helm search repo`, you can find the names of the charts in repositories you have already added:

```bash
helm repo add brigade https://brigadecore.github.io/charts

helm search repo brigade
```

Helm search uses fuzzy string matching, so you can type parts of words or phrases.

### `helm install` - Installing a Package
Use `helm install` to install a new package. At its simplest it takes two arguments: a release name and the name of the chart you want to install.

```bash
helm install happy-panda bitnami/wordpress
```

Installing a chart creates a new release object. If you want Helm to generate a release name for you, leave off the name and use `--generate-name`.

During the install, the `helm` client will print information about which resources were created, what the state of the release is, and whether there are additional configuration steps you can or should take.  

Helm installs resourcces in the following order:
-   Namespace
-   NetworkPolicy
-   ResourceQuota
-   LimitRange
-   PodSecurityPolicy
-   PodDisruptionBudget
-   ServiceAccount
-   Secret
-   SecretList
-   ConfigMap
-   StorageClass
-   PersistentVolume
-   PersistentVolumeClaim
-   CustomResourceDefinition
-   ClusterRole
-   ClusterRoleList
-   ClusterRoleBinding
-   ClusterRoleBindingList
-   Role
-   RoleList
-   RoleBinding
-   RoleBindingList
-   Service
-   DaemonSet
-   Pod
-   ReplicationController
-   ReplicaSet
-   Deployment
-   HorizontalPodAutoscaler
-   StatefulSet
-   Job
-   CronJob
-   Ingress
-   APIService

Helm doesn't wait until all the resources are running before it exits. Many charts require Docker images that are over 600MB in size, and may take a long time to install in the cluster. 

To keep track of a release's state, or to check configuration infrmation, use `helm status`.

#### Customizing the Chart Before Installing
To see what options are configurable on a chart, use `helm show values`. You can then override any of these settings in a YAML formatted file, and pass that file during installation.

```bash
echo '{mariadb.auth.database: user0db, mariadb.auth.username: user0}' > values.yaml

helm install -f values.yaml bitnami/wordpress --generate-name
```

There are two ways to pass configuration data during install:
- `--values` - Specify a YAML file with overrides. This can be specified multiple times and the rightmost file will take precedence
- `--set` - Specify overrides on the command line

If both are used, `--set` values are merged into `--values`with higher precedence. verrides specified with `--set` are persisted in a ConfigMap. Values that have been `--set` can be viewed for a given release with `helm get values release_name`. Values that have been `--set` can be cleared by running `helm upgrade` with `--reset-values` specified.

#### More Installation Methods
The `helm install` command can install from several surces:
- A chart repository
- A local chart archive `helm install chart_name chart_name.1.2.3.tgz`
- An unpacked chart directory `helm install chart_name path/to/chart_name`
- A full URL `helm install chart_name https://example.com/charts/chart_name.1.2.3.tgz`

### `helm upgrade` and `helm rollback` - Upgrading a Release and Recoverng on Failure
When a new version of a chart is released, or when you want to change the configuration of your release, you can use `helm upgrade`.

An upgrade takes an existing release and upgrades it according to the information you provide. Because Kubernetes charts can be large an dcomplex, Helm tries to perform the least invasive upgrade. It will only update things that have been changed since the last release.

```bash
helm upgrade -f panda.yaml happy-panda bitnami/wordpress
```

In the above example, the `happy-panda` release  is upgraded with the same chart but a new YAML file .

You can use `helm get values release_name` to see whether the new setting took effect.

If something does not go as planned during a release, you can rollback to a previous release using `helm rollback release_name revision_number`

```bash
helm rollback happy-panda 1
```

The above example will roll back the `happy-panda` release to its very first release version. A release version is an incremental revision. Every time an install, upgrade, or rollback happens, the revision number s incremented by 1. The first revision number is always 1. You can use `helm history revision_number` to see revision number for a certain release.

### Options for Install/Upgrade/Rollback
There are several options you can specify to customize the behaviour of Helm during an install/upgrade/rollback.
- `--timeout` - A Go duration value to wait for Kubernetes commands to complete. The default is `5m0s`
- `--wait` - Waits until all pods are in a ready state, PVCs are bound, Deployments have a minimum number (`Desired` mnus `maxUnavailable`) oof Pods in a ready state, and Services have an IP address (and Ingress if a `LoadBalancer`) before marking the release as successful. It will wait for as long as the `--timeout` value. If the timeut is reached, the release will be marked as `FAILED`
- `--no-hooks` - This skips running hooks for the command
- `--recreate-pods` (only available for `upgrade` and `rollback`, deprecated in Helm 3) - This flag will cause all pods to be recreated, with the exception of pods belonging to deplooyments.

### `helm uninstall` - Uninstalling a Release
When it's time to uninstall a release from the cluster, use `helm uninstall release_name`.

You can see all of your currently deployed releases with the `helm lst` command.

From Helm 3, deletion removes the record of the deletion. If you want to keep a deletion release record, use `helm uninstall --keep-history`.

`helm list --all` wll show all release records that Helm has retained, including records for failed or deleted items, if `--keep-history` as specified.

Because releases are now deleted by default, it is no longer possible to rollback an uninstalled resource.

### `helm repo` - Working with Repositories
Helm 3 no longer shps with a default chart repository. The `helm repo` group of commands provide commands  to add, list and remove repositories.

You can see which repositories are configured using `helm repo list`.

New repositories can be added using `helm repo add`:

```bash
helm repo add dev https://example.com/dev-charts
```

Since repositories change frequently, at any point you can make sure yoour Helm client is up to date by running `helm repo update`.

Repositories can be removed with `helm repo remove`.

### Creating your own Charts
You can create charts using the `helm create` command.

```bash
helm create deiis-workflow
```

Now there will be a chart in `./deis-workflow`. You can edit it and create your own workflows.

As you edit your chart, you can validate that it is well-formed by running `helm lint`.

When you want to package the chart for distribution, you can run the `helm package` command.

```bash
helm package deis-workflow
```

That chart can now be installed using `helm install`.

```bash
helm install deis-workflow ./deis-workflow.0.0.1.tgz
```

Charts that are packaged can be loaded into chart repositories.

## Migrating from Helm 2 to Helm 3
The main changes a user should be aware of before and during a migration are:

1. Tiller removed
	- Replaces client/server with client/library architecture
	- Security is now on a per-user basis
	- Releases are now stored as in-cluster secrets and the release object metadata has changed
	- Releases are persisted on a release namespace basis and not in the Tiller namespace any more
2. Chart repository updated
	- `helm search` now supports both local repository searches and making search queries against Artifact Hub
3. Chat apiVersion bumped to "v2" for the following specification changes:
	- Dynamically linked chart dependencies moved to `Chart.yaml`. `requirements.yaml` removed and requirements moved to dependencies
	- Library charts (helper/common charts) can now be added as dynamically linked chart dependencies
	- Charts have a `type` metadata field t define the chart to be an `application` or `library` chart. It's application by default, meaning it is renderable and installable
	- Helm 2 charts (apiVersion = v1) are still installable
4. XDG directory specification added:
	- Helm home removed and replaced with XDg diectoory specification for storing cnfiguration files
	- No longer need to initialize Helm
	- `helm init` and `helm home` removed
5. Additonal changes:
	- Helm install/set-up is simplified
		- Helm client (helm binary) only (no Tiller)
		- Run-as-is paradigm
	- `local` or `stable` repositories are not set up by default
	- `crd-install` hook removed and replaced with `crds/` directory in chart, where all CRDs defined in it will be installed before any rendering of the chart
	- `test-failure` hook annotation remved and `test-success` deprecated. Use `test` instead
	- Commands remved/replaced/added
		- `delete` -> `uninstall`: removes all release history by default (previously needed `--purge`)
		- `fetch` -> `pull`
		- `home` (removed)
		- `init` (removed)
		- `install`: requires release name or `--generate-name` argument
		- `inspect` -> `show`
		- `reset` (removed)
		- `serve` (removed)
		- `template`: `-x / --execute` argument renamed to `-s / --show-only`
		- `upgrade`: Added argument `--history-max` which limits the maximum number of revisions saved per release (0 for no limit)
	- Helm 3 Go library has undergone a lot of changes and is incompatible with the Helm 2 library
	- Release binaries are now hosted on `get.helm.sh`

A Helm 2 client can
- manage 1 to many Kubernetes clusters
- connect to 1 to many Tiller instances for a cluster

You need to be aware of this when migrating, since releases are deployed into clusters by Tiller and its namespace. As such, you have to be aware of migrating for each cluster and each Tiller instance that is managed by the Helm 2 client instance.

The recommended migration path is:
1. Back up v2 data
2. Migrate Helm v2 configuration
3. Migrate Helm v2 releases
4. When confident that Helm v3 is managing all Helm v2 data (for all clusters and all Tiller instances of the Helm v2 client instance) as expected, then clean up Helm v2 data

The migration process is automated by the Helm v3 [helm-2to3](https://github.com/helm/helm-2to3) plugin.
