---
title: "Understanding generator.yaml configuration"
description: "Understanding the ACK generator.yaml configuration file"
lead: ""
draft: false
menu:
  docs:
    parent: "contributor"
weight: 55
toc: true
---

# Understanding generator.yaml Configuration

This document describes the various configuration fields in a `generator.yaml` file that can be used to control the API inference and code generation for an ACK controller.

We will show examples of configuring specific ACK controllers to highlight various configuration options.

## Generate a resource manager package

For this section, we will use [ECR](https://docs.aws.amazon.com/AmazonECR/latest/APIReference/Welcome.html) as our example service API.

When creating a new ACK controller, after running the [`controller-bootstrap`](https://github.com/aws-controllers-k8s/controller-bootstrap) program, you will be left with a `generator.yaml` that has all inferred API resources ignored.

For the ECR controller, the `generator.yaml` file would look like this:

```yaml=
ignore:
  resource_names:
    - Repository
    - PullThroughCacheRule
```

If we ran `make build-controller SERVICE=ecr` with the above `generator.yaml` file, we would have some basic directories and files created:

```bash=
[jaypipes@thelio code-generator]$ make build-controller SERVICE=ecr
building ack-generate ... ok.
==== building ecr-controller ====
Copying common custom resource definitions into ecr
Building Kubernetes API objects for ecr
Generating deepcopy code for ecr
Generating custom resource definitions for ecr
Building service controller for ecr
Generating RBAC manifests for ecr
Running gofmt against generated code for ecr
Updating additional GitHub repository maintenance files
==== building ecr-controller release artifacts ====
Building release artifacts for ecr-v0.0.0-non-release
Generating common custom resource definitions
Generating custom resource definitions for ecr
Generating RBAC manifests for ecr
```

```
[jaypipes@thelio ecr-controller]$ tree apis/ config/ pkg/
apis/
└── v1alpha1
    ├── ack-generate-metadata.yaml
    ├── doc.go
    ├── enums.go
    ├── generator.yaml
    ├── groupversion_info.go
    └── types.go
config/
├── controller
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
├── crd
│   ├── common
│   │   ├── bases
│   │   │   ├── services.k8s.aws_adoptedresources.yaml
│   │   │   └── services.k8s.aws_fieldexports.yaml
│   │   └── kustomization.yaml
│   └── kustomization.yaml
├── default
│   └── kustomization.yaml
├── overlays
│   └── namespaced
│       ├── kustomization.yaml
│       ├── role-binding.json
│       └── role.json
└── rbac
    ├── cluster-role-binding.yaml
    ├── cluster-role-controller.yaml
    ├── kustomization.yaml
    ├── role-reader.yaml
    ├── role-writer.yaml
    └── service-account.yaml
pkg/
├── resource
│   └── registry.go
└── version
    └── version.go

11 directories, 25 files
```

To begin generating a particular resource manager, comment out the name of the resource from the ignore list and run `make build-controller SERVICE=$SERVICE`.

```yaml=
ignore:
  resource_names:
>   #- Repository
    - PullThroughCacheRule
```

After doing so, the resource manager for `Repository` resources will have been generated in the `ecr-controller` source code repository.

```
[jaypipes@thelio ecr-controller]$ tree apis/ config/ pkg/
apis/
└── v1alpha1
    ├── ack-generate-metadata.yaml
    ├── doc.go
    ├── enums.go
    ├── generator.yaml
    ├── groupversion_info.go
    ├── repository.go
    ├── types.go
    └── zz_generated.deepcopy.go
config/
├── controller
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
├── crd
│   ├── bases
│   │   └── ecr.services.k8s.aws_repositories.yaml
│   ├── common
│   │   ├── bases
│   │   │   ├── services.k8s.aws_adoptedresources.yaml
│   │   │   └── services.k8s.aws_fieldexports.yaml
│   │   └── kustomization.yaml
│   └── kustomization.yaml
├── default
│   └── kustomization.yaml
├── overlays
│   └── namespaced
│       ├── kustomization.yaml
│       ├── role-binding.json
│       └── role.json
└── rbac
    ├── cluster-role-binding.yaml
    ├── cluster-role-controller.yaml
    ├── kustomization.yaml
    ├── role-reader.yaml
    ├── role-writer.yaml
    └── service-account.yaml
pkg/
├── resource
│   ├── registry.go
│   └── repository
│       ├── delta.go
│       ├── descriptor.go
│       ├── identifiers.go
│       ├── manager_factory.go
│       ├── manager.go
│       ├── references.go
│       ├── resource.go
│       ├── sdk.go
│       └── tags.go
└── version
    └── version.go

13 directories, 37 files
```

Note the new files under `pkg/resource/repository/`, `apis/v1alpha1/repository.go` and `config/crd/bases/`.  These files represent the Go type for the generated `Repository` custom resource definition (CRD), the resource manager package and the YAML representation for the CRD, respectively.

## Renaming things

Why might we want to rename fields or resources? Generally, there are two reasons for this:
 - reducing stutter in the input shape
 - correcting instances where a field is named differently in the input and output shapes

The first reason is to reduce "stutter" (or redundancy) in naming. For example, the ECR [`Repository`](https://github.com/aws-controllers-k8s/ecr-controller/blob/cefc45b65e1a560b9666a1544f26f955c6f84d36/generator.yaml#L2) resource has a field called `RepositoryName`.  This field is redundantly named because the resource itself is called `Repository`.  Every Kubernetes object has a `Metadata.Name` field and we like to align resource "name fields" with this simple `Name` moniker.

For this example, let's go ahead and "destutter" the `RepositoryName` field. To do this, we use the `renames` configuration option, specifying the input and output shapes and their members that we want to rename:

```yaml=
ignore:
  resource_names:
    #- Repository
    - PullThroughCacheRule
resources:
  Repository:
>   renames:
>     operations:
>       CreateRepository:
>         input_fields:
>           RepositoryName: Name
>       DeleteRepository:
>         input_fields:
>           RepositoryName: Name
>       DescribeRepositories:
>         input_fields:
>           RepositoryName: Name
```

> 📝 Note that we must tell the code generator which fields to rename in the input shapes for each API operation that the resource manager will call.  In the case of ECR `Repository` resources, the resource manager calls the `CreateRepository`, `DeleteRepository` and `DescribeRepositories` API calls and so we need specify the `RepositoryName` member field in each of those input shapes should be renamed to `Name`.

After calling `make build-controller SERVICE=ecr`, we see the above generator configuration items produced the following diff:

```diff=
diff --git a/apis/v1alpha1/ack-generate-metadata.yaml b/apis/v1alpha1/ack-generate-metadata.yaml
index e34e029..f214b43 100755
--- a/apis/v1alpha1/ack-generate-metadata.yaml
+++ b/apis/v1alpha1/ack-generate-metadata.yaml
@@ -1,13 +1,13 @@
 ack_generate_info:
-  build_date: "2022-11-09T20:15:42Z"
+  build_date: "2022-11-09T20:16:52Z"
   build_hash: 5ee0ac052c54f008dff50f6f5ebb73f2cf3a0bd7
   go_version: go1.18.1
   version: v0.20.1-4-g5ee0ac0
-api_directory_checksum: 0a514bef9cff983f9fe28f080d85725ccf578060
+api_directory_checksum: 84fb59a0991980da922a385f585111a1ff784d82
 api_version: v1alpha1
 aws_sdk_go_version: v1.44.93
 generator_config_info:
-  file_checksum: 87446926d73abae9355e6328eb7f8f668b16b18e
+  file_checksum: a383007f82a686dc544879792dde7b091aeededa
   original_file_name: generator.yaml
 last_modification:
   reason: API generation
diff --git a/apis/v1alpha1/generator.yaml b/apis/v1alpha1/generator.yaml
index cb7045a..ed0130f 100644
--- a/apis/v1alpha1/generator.yaml
+++ b/apis/v1alpha1/generator.yaml
@@ -2,3 +2,16 @@ ignore:
   resource_names:
     #- Repository
     - PullThroughCacheRule
+resources:
+  Repository:
+    renames:
+      operations:
+        CreateRepository:
+          input_fields:
+            RepositoryName: Name
+        DeleteRepository:
+          input_fields:
+            RepositoryName: Name
+        DescribeRepositories:
+          input_fields:
+            RepositoryName: Name
diff --git a/apis/v1alpha1/repository.go b/apis/v1alpha1/repository.go
index c226d4f..fc6165d 100644
--- a/apis/v1alpha1/repository.go
+++ b/apis/v1alpha1/repository.go
@@ -35,15 +35,15 @@ type RepositorySpec struct {
 	// be overwritten. If IMMUTABLE is specified, all image tags within the repository
 	// will be immutable which will prevent them from being overwritten.
 	ImageTagMutability *string `json:"imageTagMutability,omitempty"`
-	// The Amazon Web Services account ID associated with the registry to create
-	// the repository. If you do not specify a registry, the default registry is
-	// assumed.
-	RegistryID *string `json:"registryID,omitempty"`
 	// The name to use for the repository. The repository name may be specified
 	// on its own (such as nginx-web-app) or it can be prepended with a namespace
 	// to group the repository into a category (such as project-a/nginx-web-app).
 	// +kubebuilder:validation:Required
-	RepositoryName *string `json:"repositoryName"`
+	Name *string `json:"name"`
+	// The Amazon Web Services account ID associated with the registry to create
+	// the repository. If you do not specify a registry, the default registry is
+	// assumed.
+	RegistryID *string `json:"registryID,omitempty"`
 	// The metadata that you apply to the repository to help you categorize and
 	// organize them. Each tag consists of a key and an optional value, both of
 	// which you define. Tag keys can have a maximum character length of 128 characters,
diff --git a/apis/v1alpha1/zz_generated.deepcopy.go b/apis/v1alpha1/zz_generated.deepcopy.go
index 93919be..88dd4c0 100644
--- a/apis/v1alpha1/zz_generated.deepcopy.go
+++ b/apis/v1alpha1/zz_generated.deepcopy.go
@@ -421,13 +421,13 @@ func (in *RepositorySpec) DeepCopyInto(out *RepositorySpec) {
 		*out = new(string)
 		**out = **in
 	}
-	if in.RegistryID != nil {
-		in, out := &in.RegistryID, &out.RegistryID
+	if in.Name != nil {
+		in, out := &in.Name, &out.Name
 		*out = new(string)
 		**out = **in
 	}
-	if in.RepositoryName != nil {
-		in, out := &in.RepositoryName, &out.RepositoryName
+	if in.RegistryID != nil {
+		in, out := &in.RegistryID, &out.RegistryID
 		*out = new(string)
 		**out = **in
 	}
diff --git a/config/crd/bases/ecr.services.k8s.aws_repositories.yaml b/config/crd/bases/ecr.services.k8s.aws_repositories.yaml
index 438785e..9657569 100644
--- a/config/crd/bases/ecr.services.k8s.aws_repositories.yaml
+++ b/config/crd/bases/ecr.services.k8s.aws_repositories.yaml
@@ -61,17 +61,17 @@ spec:
                   all image tags within the repository will be immutable which will
                   prevent them from being overwritten.
                 type: string
-              registryID:
-                description: The Amazon Web Services account ID associated with the
-                  registry to create the repository. If you do not specify a registry,
-                  the default registry is assumed.
-                type: string
-              repositoryName:
+              name:
                 description: The name to use for the repository. The repository name
                   may be specified on its own (such as nginx-web-app) or it can be
                   prepended with a namespace to group the repository into a category
                   (such as project-a/nginx-web-app).
                 type: string
+              registryID:
+                description: The Amazon Web Services account ID associated with the
+                  registry to create the repository. If you do not specify a registry,
+                  the default registry is assumed.
+                type: string
               tags:
                 description: The metadata that you apply to the repository to help
                   you categorize and organize them. Each tag consists of a key and
@@ -92,7 +92,7 @@ spec:
                   type: object
                 type: array
             required:
-            - repositoryName
+            - name
             type: object
           status:
             description: RepositoryStatus defines the observed state of Repository
diff --git a/generator.yaml b/generator.yaml
index cb7045a..ed0130f 100644
--- a/generator.yaml
+++ b/generator.yaml
@@ -2,3 +2,16 @@ ignore:
   resource_names:
     #- Repository
     - PullThroughCacheRule
+resources:
+  Repository:
+    renames:
+      operations:
+        CreateRepository:
+          input_fields:
+            RepositoryName: Name
+        DeleteRepository:
+          input_fields:
+            RepositoryName: Name
+        DescribeRepositories:
+          input_fields:
+            RepositoryName: Name
diff --git a/helm/crds/ecr.services.k8s.aws_repositories.yaml b/helm/crds/ecr.services.k8s.aws_repositories.yaml
index 438785e..9657569 100644
--- a/helm/crds/ecr.services.k8s.aws_repositories.yaml
+++ b/helm/crds/ecr.services.k8s.aws_repositories.yaml
@@ -61,17 +61,17 @@ spec:
                   all image tags within the repository will be immutable which will
                   prevent them from being overwritten.
                 type: string
-              registryID:
-                description: The Amazon Web Services account ID associated with the
-                  registry to create the repository. If you do not specify a registry,
-                  the default registry is assumed.
-                type: string
-              repositoryName:
+              name:
                 description: The name to use for the repository. The repository name
                   may be specified on its own (such as nginx-web-app) or it can be
                   prepended with a namespace to group the repository into a category
                   (such as project-a/nginx-web-app).
                 type: string
+              registryID:
+                description: The Amazon Web Services account ID associated with the
+                  registry to create the repository. If you do not specify a registry,
+                  the default registry is assumed.
+                type: string
               tags:
                 description: The metadata that you apply to the repository to help
                   you categorize and organize them. Each tag consists of a key and
@@ -92,7 +92,7 @@ spec:
                   type: object
                 type: array
             required:
-            - repositoryName
+            - name
             type: object
           status:
             description: RepositoryStatus defines the observed state of Repository
diff --git a/pkg/resource/repository/delta.go b/pkg/resource/repository/delta.go
index a15d260..57b54df 100644
--- a/pkg/resource/repository/delta.go
+++ b/pkg/resource/repository/delta.go
@@ -77,6 +77,13 @@ func newResourceDelta(
 			delta.Add("Spec.ImageTagMutability", a.ko.Spec.ImageTagMutability, b.ko.Spec.ImageTagMutability)
 		}
 	}
+	if ackcompare.HasNilDifference(a.ko.Spec.Name, b.ko.Spec.Name) {
+		delta.Add("Spec.Name", a.ko.Spec.Name, b.ko.Spec.Name)
+	} else if a.ko.Spec.Name != nil && b.ko.Spec.Name != nil {
+		if *a.ko.Spec.Name != *b.ko.Spec.Name {
+			delta.Add("Spec.Name", a.ko.Spec.Name, b.ko.Spec.Name)
+		}
+	}
 	if ackcompare.HasNilDifference(a.ko.Spec.RegistryID, b.ko.Spec.RegistryID) {
 		delta.Add("Spec.RegistryID", a.ko.Spec.RegistryID, b.ko.Spec.RegistryID)
 	} else if a.ko.Spec.RegistryID != nil && b.ko.Spec.RegistryID != nil {
@@ -84,13 +91,6 @@ func newResourceDelta(
 			delta.Add("Spec.RegistryID", a.ko.Spec.RegistryID, b.ko.Spec.RegistryID)
 		}
 	}
-	if ackcompare.HasNilDifference(a.ko.Spec.RepositoryName, b.ko.Spec.RepositoryName) {
-		delta.Add("Spec.RepositoryName", a.ko.Spec.RepositoryName, b.ko.Spec.RepositoryName)
-	} else if a.ko.Spec.RepositoryName != nil && b.ko.Spec.RepositoryName != nil {
-		if *a.ko.Spec.RepositoryName != *b.ko.Spec.RepositoryName {
-			delta.Add("Spec.RepositoryName", a.ko.Spec.RepositoryName, b.ko.Spec.RepositoryName)
-		}
-	}
 	if !reflect.DeepEqual(a.ko.Spec.Tags, b.ko.Spec.Tags) {
 		delta.Add("Spec.Tags", a.ko.Spec.Tags, b.ko.Spec.Tags)
 	}
diff --git a/pkg/resource/repository/resource.go b/pkg/resource/repository/resource.go
index e15d755..a2dd27e 100644
--- a/pkg/resource/repository/resource.go
+++ b/pkg/resource/repository/resource.go
@@ -88,7 +88,7 @@ func (r *resource) SetIdentifiers(identifier *ackv1alpha1.AWSIdentifiers) error
 	if identifier.NameOrID == "" {
 		return ackerrors.MissingNameIdentifier
 	}
-	r.ko.Spec.RepositoryName = &identifier.NameOrID
+	r.ko.Spec.Name = &identifier.NameOrID
 
 	f2, f2ok := identifier.AdditionalKeys["registryID"]
 	if f2ok {
diff --git a/pkg/resource/repository/sdk.go b/pkg/resource/repository/sdk.go
index 4366244..61a0053 100644
--- a/pkg/resource/repository/sdk.go
+++ b/pkg/resource/repository/sdk.go
@@ -132,9 +132,9 @@ func (rm *resourceManager) sdkFind(
 			ko.Status.ACKResourceMetadata.ARN = &tmpARN
 		}
 		if elem.RepositoryName != nil {
-			ko.Spec.RepositoryName = elem.RepositoryName
+			ko.Spec.Name = elem.RepositoryName
 		} else {
-			ko.Spec.RepositoryName = nil
+			ko.Spec.Name = nil
 		}
 		if elem.RepositoryUri != nil {
 			ko.Status.RepositoryURI = elem.RepositoryUri
@@ -158,7 +158,7 @@ func (rm *resourceManager) sdkFind(
 func (rm *resourceManager) requiredFieldsMissingFromReadManyInput(
 	r *resource,
 ) bool {
-	return r.ko.Spec.RepositoryName == nil
+	return r.ko.Spec.Name == nil
 
 }
 
@@ -172,9 +172,9 @@ func (rm *resourceManager) newListRequestPayload(
 	if r.ko.Spec.RegistryID != nil {
 		res.SetRegistryId(*r.ko.Spec.RegistryID)
 	}
-	if r.ko.Spec.RepositoryName != nil {
+	if r.ko.Spec.Name != nil {
 		f3 := []*string{}
-		f3 = append(f3, r.ko.Spec.RepositoryName)
+		f3 = append(f3, r.ko.Spec.Name)
 		res.SetRepositoryNames(f3)
 	}
 
@@ -253,9 +253,9 @@ func (rm *resourceManager) sdkCreate(
 		ko.Status.ACKResourceMetadata.ARN = &arn
 	}
 	if resp.Repository.RepositoryName != nil {
-		ko.Spec.RepositoryName = resp.Repository.RepositoryName
+		ko.Spec.Name = resp.Repository.RepositoryName
 	} else {
-		ko.Spec.RepositoryName = nil
+		ko.Spec.Name = nil
 	}
 	if resp.Repository.RepositoryUri != nil {
 		ko.Status.RepositoryURI = resp.Repository.RepositoryUri
@@ -298,8 +298,8 @@ func (rm *resourceManager) newCreateRequestPayload(
 	if r.ko.Spec.RegistryID != nil {
 		res.SetRegistryId(*r.ko.Spec.RegistryID)
 	}
-	if r.ko.Spec.RepositoryName != nil {
-		res.SetRepositoryName(*r.ko.Spec.RepositoryName)
+	if r.ko.Spec.Name != nil {
+		res.SetRepositoryName(*r.ko.Spec.Name)
 	}
 	if r.ko.Spec.Tags != nil {
 		f5 := []*svcsdk.Tag{}
@@ -362,8 +362,8 @@ func (rm *resourceManager) newDeleteRequestPayload(
 	if r.ko.Spec.RegistryID != nil {
 		res.SetRegistryId(*r.ko.Spec.RegistryID)
 	}
-	if r.ko.Spec.RepositoryName != nil {
-		res.SetRepositoryName(*r.ko.Spec.RepositoryName)
+	if r.ko.Spec.Name != nil {
+		res.SetRepositoryName(*r.ko.Spec.Name)
 	}
 
 	return res, nil
```

You will note that there were changes made to the `repository.go` file, the `pkg/resource/repository/sdk.go`.

### Renaming things with different names in input and output shapes

The second reason we might need to rename a field is when the same field goes by different names in the *shapes* (i.e., expected syntax) of the input and output.  An hypothetical example of this might be a field that is called `EnableEncryption` in an input shape and `EncryptionEnabled` in an output shape.  In order to inform the code generator that these fields are actually the same, we would rename one of the fields to match the other.

[**_TODO_** this needs a concrete example of renaming with both `input_fields` and `output_fields`]


## Ignoring things

Sometimes you want to instruct the code generator to simply ignore a particular API Operation, or a particular field in an API Shape.  See [here](https://github.com/aws-controllers-k8s/s3-controller/pull/89) for a real world motivating example of such a need.

You will use the [`ignore:`][ignore-config] block of configuration options to do this.

[ignore-config]: https://github.com/aws-controllers-k8s/code-generator/blob/f6dd767f12429832bc7b4321fb7b763a9fa997c7/pkg/config/config.go#L50-L66

To ignore a specific field in an API Shape, you can list the field via fieldpath in the `ignore.fieldpaths` configuration option.

An [example][ignore-fieldpaths-example] of this can be found in the S3 controller's `generator.yaml` file:

[ignore-fieldpaths-example]: https://github.com/aws-controllers-k8s/s3-controller/blob/9adb7703fa9e8c422a583ec1c8da35ecb21c8917/generator.yaml#L8-L12

```yaml=
ignore:
  field_paths:
    # We cannot support MFA, so if it is set we cannot unset
    - "VersioningConfiguration.MFADelete"
    # This subfield struct has no members...
    - "NotificationConfiguration.EventBridgeConfiguration"
```

When you specify a field path in `ignore.field_paths`, the code generator will skip over that field when inferring custom resource definition `Spec` and `Status` structures.

## Tags

*Most* resources in AWS service APIs can have one or more tags associated with them.  Tags are *typically* simple string key/value pairs; however, the representation of tags across different AWS service APIs is not consistent.  Some APIs use a `map[string]string` to represent tags. Others use a `[]struct{}` where the struct has a `Key` and a `Value` field. Others use more complex structures.

### Telling ACK code generator that a resource does not support tags

There are some API resources that *do not* support tags at all, and we want a way to skip the generation of code that handles tagging for those resources. By default, for all resources, ACK generates some code that handles conversion between the ACK standard representation of tags (i.e., `map[string]string`) and the AWS service-specific representation of tags (e.g., `[]struct{}`, etc).

If you attempt to generate a resource manager for a resource that does not support tags, you will receive an error from the code generator.  ECR's [`PassThroughCacheRule`](https://docs.aws.amazon.com/AmazonECR/latest/APIReference/API_CreatePullThroughCacheRule.html) is an example of a resource that does not support tags. If we unignore the `PassThroughCacheRule` resource in the ECR controller's [`generator.yaml`](https://github.com/aws-controllers-k8s/ecr-controller/blob/cefc45b65e1a560b9666a1544f26f955c6f84d36/generator.yaml#L78) file and regenerate the controller, we will stumble upon this error:

```
[jaypipes@thelio code-generator]$ make build-controller SERVICE=ecr
building ack-generate ... ok.
==== building ecr-controller ====
Copying common custom resource definitions into ecr
Building Kubernetes API objects for ecr
Generating deepcopy code for ecr
Generating custom resource definitions for ecr
Building service controller for ecr
Error: template: /home/jaypipes/go/src/github.com/aws-controllers-k8s/code-generator/templates/pkg/resource/manager.go.tpl:282:20: executing "/home/jaypipes/go/src/github.com/aws-controllers-k8s/code-generator/templates/pkg/resource/manager.go.tpl" at <.CRD.GetTagField>: error calling GetTagField: tag field path Tags does not exist inside PullThroughCacheRule crd
make: *** [Makefile:41: build-controller] Error 1
```

To fix this error, we used the `tags.ignore` configuration option in [`generator.yaml`](https://github.com/aws-controllers-k8s/ecr-controller/blob/cefc45b65e1a560b9666a1544f26f955c6f84d36/generator.yaml#L78):

```yaml=
ignore:
  resource_names:
    #- Repository
    #- PullThroughCacheRule
resources:
  Repository:
    renames:
      operations:
        CreateRepository:
          input_fields:
            RepositoryName: Name
        DeleteRepository:
          input_fields:
            RepositoryName: Name
        DescribeRepositories:
          input_fields:
            RepositoryName: Name
  PullThroughCacheRule:
    fields:
      ECRRepositoryPrefix:
        is_primary_key: true
 >  tags:
 >    ignore: true
```

## Resource configuration

### Understanding resource identifying fields

All resources in the AWS world have one or more fields that serve as primary key identifiers. Most people are familiar with the `ARN` fields that most modern AWS resources have. However, the `ARN` field is not the only field that can serve as a primary key for a resource. ACK's code generator reads an API model file and attempts to determine which fields on a resource can be used to uniquely identify that resource. Sometimes, though, the code generator needs to be instructed which field or fields comprise this primary key.  [See below](#Field-level-configuration-of-identifying-fields) for an example from ECR's `PullThroughCacheRule`.

There are resource-level and field-level configuration options that inform the code generator about identifying fields.

#### Resource-level configuration of identifying fields

The `resources[$resource].is_arn_primary_key` configuration option is a boolean, defaulting to `false` that instructs the code generator to use the `ARN` field when calling the "ReadOne" (i.e., "Describe" or "Get") operation for that resource. When `false`, the code generator will look for "identifier fields" with field names such as `ID` or `Name` (along with variants that include the resource name as a prefix, e.g., "BucketName").

Use the `is_arn_primary_key=true` configuration option *when the resource has no other identifying fields*. An example of this is SageMaker's `ModelPackage` resource that has no `Name` or `ID` field and can only be identified via an `ARN` field:

```yaml=
resources:
  ModelPackage:
    is_arn_primary_key: true
```

*[NOTE(jaypipes): Probably want to reevaluate this particular config option and use the field-centric is_primary_key option instead...]*

#### Field-level configuration of identifying fields

Sometimes a resource's primary key field is non-obvious (like `Name` or `ID`). Use the `resources[$resource]fields[$field].is_primary_key` configuration option to tell the code generator about these fields.

An example here is ECR's [`PullThroughCacheRule`](https://github.com/aws-controllers-k8s/ecr-controller/blob/cefc45b65e1a560b9666a1544f26f955c6f84d36/generator.yaml#L57) resource, which [has a primary key field](https://docs.aws.amazon.com/AmazonECR/latest/APIReference/API_CreatePullThroughCacheRule.html) called `ECRRepositoryPrefix`:

```yaml=
resources:
  PullThroughCacheRule:
    fields:
      ECRRepositoryPrefix:
        is_primary_key: true
```

*[NOTE(jljaco):  If we discard `is_arn_primary_key` in favor of only `is_primary_key`, this sub-section should be moved into the `Field Configuration` section]*

### Correcting exception codes

An ACK controller needs to understand which HTTP exception code means "this resource was not found"; otherwise, the controller's logic that determines whether to create or update a resource falls apart.
 
For the majority of AWS service APIs, the ACK code generator can figure out which HTTP exception codes map to which HTTP fault behaviours. However, some AWS service API model definitions do not include exception metadata. Other service API models include straight-up incorrect information that does not match what the actual AWS service returns.

To address these issues, you can use the `resources[$resource].exceptions` configuration block.

An example of an API model that does not indicate the exception code representing a resource not found is DynamoDB. When calling DynamoDB's [`DescribeTable`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_DescribeTable.html) API call with a table name that does not exist, you will get back a `400` error code instead of `404` and the exception code string is `ResourceNotFoundException`.

To tell the ACK code generator how to deal with this, use the `exceptions` [configuration option](https://github.com/aws-controllers-k8s/code-generator/blob/160b839fe09dd7e1f321e094604ffc3b6ae2a285/pkg/config/resource.go#L251):

```yaml=
resources:
  Table:
    exceptions:
      errors:
        404:
          code: ResourceNotFoundException
```

This configuration instructs the code generator to [produce code](https://github.com/aws-controllers-k8s/dynamodb-controller/blob/eb1405d3d10050c8a866dc0e2dc0ec72c8213886/pkg/resource/table/sdk.go#L79-L84) that looks for `ResourceNotFoundException` in the error response of the API call and interprets it properly as a `404` or "resource not found" error.

#### Specifying terminal codes to indicate terminal state

An `ACK.Terminal` `Condition` is placed on a custom resource (inside of its `Status`) when the controller realizes that, without the user changing the resource's `Spec`, the resource will not be able to be reconciled (i.e., the desired state will never match the actual state).

When an ACK controller gets a response back from an AWS service containing an error code, the controller evaluates whether that error code should result in the `ACK.Terminal` `Condition` being placed on the resource. Examples of these "terminal codes" are things such as: 

- improper input being supplied
- a duplicate resource already existing
- conflicting input values

AWS service API responses having a `4XX` HTTP status code will have a corresponding exception string code (e.g., `InvalidParameterValue` or `EntityExistsException`). Use the `resources[$resource].exceptions.terminal_codes` configuration option to tell the code generation which of these exception string codes it should consider to be a *terminal state* for the resource.

Here is [an example from the RDS controller](https://github.com/aws-controllers-k8s/rds-controller/blob/f97b026cdd72e222390f42a18770fb0de49c3b41/generator.yaml#L97), where we indicate the set of exception string code that will set the resource into a *terminal state*:

```yaml=
resources:
  DBCluster:
    exceptions:
      terminal_codes:
        - DBClusterQuotaExceededFault
        - DBSubnetGroupDoesNotCoverEnoughAZs
        - InsufficientStorageClusterCapacity
        - InvalidParameter
        - InvalidParameterValue
        - InvalidParameterCombination
        - InvalidSubnet
        - StorageQuotaExceeded
```

### Controlling reconciliation and requeue logic

By default, an ACK controller will requeue a resource for future reconciliation only when the resource is in some transitional state.

For example, when you create an RDS [`DBInstance`](https://github.com/aws-controllers-k8s/rds-controller/blob/f97b026cdd72e222390f42a18770fb0de49c3b41/generator.yaml#L184) resource, the resource initially goes into a `CREATING` transitional state and then eventually will arrive at an `AVAILABLE` state. When the RDS controller for ACK initially creates the RDS `DBInstance` resource, it calls the RDS [`CreateDBInstance`](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_CreateDBInstance.html) API call, sees the state of the DB instance is `CREATING`, adds an `ACK.ResourceSynced=False` `Condition` to the resource and *requeues* the resource to be processed again in a few seconds.

When the resource is processed in the next reconciliation loop, the controller calls the [`DescribeDBInstance`](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_DeleteDBInstance.html) API endpoint and checks to see if the DB instance is in the `AVAILABLE` state. If it is not, then the controller requeues the resource again. If it is in the `AVAILABLE` state, then the controller sets the `ACK.ResourceSynced` `Condition` to `True`, which is the indication to the ACK runtime that the resource should *not* be requeued.

Sometimes, you may want to have the ACK controller requeue certain resources *even after a successful reconciliation loop that leaves the resource in the `ACK.ResourceSynced=True` state*. If this is the case, you should use the `resources[$resource].reconcile.requeue_on_success_seconds` configuration option. The value of this option should be the amount of time (in seconds) after which the reconciler should requeue the resource.

Here is an example of this configuration option as used in the SageMaker controller's [`NotebookInstance` resource](https://github.com/aws-controllers-k8s/sagemaker-controller/blob/c77c6def970cf80322bacaa6aa5ff58dde671dbf/generator.yaml#L595):

```yaml=
resources:
  NotebookInstance:
    # Resource state/status can be modified in Sagemaker Console
    # Need to reconcile to catch these state/status changes
    reconcile: 
      requeue_on_success_seconds: 60
```

We set this `requeue_on_success_seconds` value to `60` here because the values of various fields in this Sagemaker resource tend to change often and we want the `Status` section of our custom resource to contain values that are fresher than the default requeue period (10 hours as of this writing).

### Including additional printer columns

**TODO(jljaco)**

## Field configuration

When `ack-generate` first [infers the definition of a resource][api-inference] from the AWS API model, it collects the various member fields of a resource. This documentation section discusses the configuration options that instruct the code generator about a particular resource field.

[api-inference]: https://aws-controllers-k8s.github.io/community/docs/contributor-docs/api-inference/

### Manually marking a field as belonging to the resource `Status` struct

During API inference, `ack-generate` automatically determines which fields belong in the custom resource definition's `Spec` or `Status` struct. Fields that can be modified by the user go in the `Spec` and fields that cannot be modified go in the `Status`.

Use the `resources[$resource].fields[$field].is_read_only` configuration option to override whether a field should go in the `Status` struct.

Here is an example from the Lambda controller's [generator.yaml](https://github.com/aws-controllers-k8s/lambda-controller/blob/2ee7a2969ee23c900a18f6265a272669b058b62e/generator.yaml#L52-L53) file that instructs the code generator to treat the `LayerStatuses` field as a read-only field (and thus should belong in the `Status` struct for the Function resource):

```yaml
resources:
  Function:
    fields:
      LayerStatuses:
        is_read_only: true
```

Typically, you will see this configuration option used for fields that have two different Go types representing the modifiable version of the field and the non-modifiable version of the field (as is the case for a Lambda Function's Layers information) or when you need to create a custom field.

### Marking a field as required

If an AWS API model file marks a particular member field as required, `ack-generate` will usually infer that the associated custom resource field is required. Sometimes, however, you may want to override whether or not a field should be required. Use the `resources[$resource].fields[$field].is_required` configuration option to do so.

Here is an example from the EC2 controller's [`generator.yaml`](https://github.com/aws-controllers-k8s/ec2-controller/blob/3b1ba705df02f9b7db1a8079ed7729af7a4f213a/generator.yaml#L206-L209) file that instructs the code generator to treat the Instance custom resource's MinCount and MaxCount fields as **not required**, even though the API model definition marks these fields as required in the Create Operation's Input shape.

NOTE: The reason for this is because the EC2 controller only deals with single Instance resources, not batches of instances

```yaml
resources:
  Instance:
    fields:
      MaxCount:
        is_required: false
      MinCount:
        is_required: false
```

### Controlling a field's Go type

Use the `resources[$resource].fields[$field].type` configuration option to override a field's Go type. You will typically use this configuration option for custom fields that are not inferred by `ack-generate` by looking at the AWS API model definition.

An example of this is the Policies field for a Role custom resource definition in the IAM controller. The IAM controller uses some custom hook code to allow a Kubernetes user to specify one or more Policy ARNs for a Role simply by specifying `Spec.Policies`. To define this custom field as a list of string pointers, the IAM controller's [`generator.yaml`](https://github.com/aws-controllers-k8s/iam-controller/blob/b6a62ca48c1aea7ab78e62fb3ad6845335c1f1c2/generator.yaml#L164-L168) file uses the following:

```yaml
resources:
  Role:
    fields:
      # In order to support attaching zero or more policies to a Role, we use
      # custom update code path code that uses the Attach/DetachGroupPolicy API
      # calls to manage the set of PolicyARNs attached to this Role.
      Policies:
        type: "[]*string"
```

### Controlling how a field's values are compared

Use the `resources[$resource].fields[$field].compare` configuration option to control how the value of a field is compared between two resources. This configuration option has two boolean subfields, `is_ignored` and `nil_equals_zero_value` (TODO(jljaco): `nil_equals_zero_value` not yet implemented or used).

#### Marking a field as ignored

Use the `is_ignored` subfield to instruct the code generator to exclude this particular field from automatic value comparisons when building the `Delta` struct that compares two resources.

Typically, you will want to mark a field as ignored for comparison operations because the Go type of the field does not natively support deterministic equality operations. For example, a slice of `Tag` structs where the code generator does not know how to sort the slice means that the default `reflect.DeepEqual` call will produce non-deterministic results.  These types of fields you will want to mark with `compare.is_ignored: true` and include a custom comparison function using the `delta_pre_compare` hook, <a name="#delta_pre_compare_example_1"></a>as this example from the IAM controller's [`generator.yaml`](https://github.com/aws-controllers-k8s/iam-controller/blob/b6a62ca48c1aea7ab78e62fb3ad6845335c1f1c2/generator.yaml#L172-L174) does for the Role resource:

```yaml
  Role:
    hooks:
      delta_pre_compare:
        code: compareTags(delta, a, b)
    fields:
      Tags:
        compare:
          is_ignored: true
```

### Mutable vs. immutable fields

Use the `resources[$resource].fields[$field].is_immutable` configuration option to mark a field as immutable -- meaning the user cannot update the field after initially setting its value.

A good example of the use of `is_immutable` comes from the RDS controller's DBInstance resource's [AvailabilityZone field](https://github.com/aws-controllers-k8s/rds-controller/blob/f8b5d69f822bfc809cbfa25ef7ad60b58a4af22e/generator.yaml#L209-L212):

```yaml=
resources:
  DBInstance:
    fields:
      AvailabilityZone:
        late_initialize: {}
        is_immutable: true
```

In the case of an DBInstance resource, once the AvailabilityZone field is set by the user, it cannot be modified.

By telling the code generator that this field is immutable, it will generate code in the `sdk.go` file that [checks for whether a user has modified any immutable fields](https://github.com/aws-controllers-k8s/rds-controller/blob/f8b5d69f822bfc809cbfa25ef7ad60b58a4af22e/pkg/resource/db_instance/sdk.go#L2768) and set a Condition on the resource if so:



```go=
// sdkUpdate patches the supplied resource in the backend AWS service API and
// returns a new resource with updated fields.
func (rm *resourceManager) sdkUpdate(
	ctx context.Context,
	desired *resource,
	latest *resource,
	delta *ackcompare.Delta,
) (updated *resource, err error) {
	rlog := ackrtlog.FromContext(ctx)
	exit := rlog.Trace("rm.sdkUpdate")
	defer func() {
		exit(err)
	}()
	if immutableFieldChanges := rm.getImmutableFieldChanges(delta); len(immutableFieldChanges) > 0 {
		msg := fmt.Sprintf("Immutable Spec fields have been modified: %s", strings.Join(immutableFieldChanges, ","))
		return nil, ackerr.NewTerminalError(fmt.Errorf(msg))
	}
    ...
}
```

### Controlling where a field's definition comes from

During API inference, `ack-generate` inspects the AWS service API model definition and discovers resource fields by looking at the Input and Output shapes of the `Create` API call for that resource.  Members of the Input shape will go in the `Spec` and members of the Output shape *that are not also in the Input shape* will go into the `Status`.

This works for a majority of the field definitions, however sometimes you want to "grab a field" from a different location (i.e., other than either the Input or Output shapes of the `Create` API call).

Each Resource typically also has a `ReadOne` Operation. The ACK service controller will call this `ReadOne` Operation to get the latest observed state of a particular resource in the backend AWS API service.  The service controller sets the observed Resource's `Spec` and `Status` fields from the Output shape of the `ReadOne` Operation.  The code generator is responsible for producing the Go code that performs these "setter" methods on the Resource.

The way the code generator determines how to set the `Spec` or `Status` fields from the Output shape's member fields is by looking at the data type of the `Spec` or `Status` field with the same name as the Output shape's member field.

Importantly, in producing this "setter" Go code the code generator **assumes that the data types (Go types) in the source (the Output shape's member field) and target (the Spec or Status field) are the same**.

There are some APIs, however, where the Go type of the field in the `Create` Operation's Input shape is actually different from the same-named field in the `ReadOne` Operation's Output shape.  A good example of this is the Lambda [`CreateFunction`](https://docs.aws.amazon.com/lambda/latest/dg/API_CreateFunction.html) API call, which has a `Code` member of its Input shape that looks like this:

```json
"Code": {
   "ImageUri": "string",
   "S3Bucket": "string",
   "S3Key": "string",
   "S3ObjectVersion": "string",
   "ZipFile": blob
},
```

The [`GetFunction`](https://docs.aws.amazon.com/lambda/latest/dg/API_GetFunction.html) API call's Output shape has a same-named field called `Code` in it, but this field looks like this:

```json
"Code": {
   "ImageUri": "string",
   "Location": "string",
   "RepositoryType": "string",
   "ResolvedImageUri": "string"
},
```

This presents a conundrum to the ACK code generator, which, as noted above, assumes the data types of same-named fields in the `Create` Operation's Input shape and `ReadOne` Operation's Output shape are the same.

Use the `resources[$resource].fields[$field].from` configuration option to handle these situations.

For the Lambda `Function` Resource's `Code` field, we can inform the code generator to create three new `Status` fields (read-only) from the `Location`, `RepositoryType` and `ResolvedImageUri` fields in the `Code` member of the [`ReadOne`](https://docs.aws.amazon.com/lambda/latest/dg/API_GetFunction.html) Operation's Output shape:

```yaml
resources:
  Function:
    fields:
      CodeLocation:
        is_read_only: true
        from:
          operation: GetFunction
          path: Code.Location
      CodeRepositoryType:
        is_read_only: true
        from:
          operation: GetFunction
          path: Code.RepositoryType
      CodeRegisteredImageURI:
        is_read_only: true
        from:
          operation: GetFunction
          path: Code.RegisteredImageUri
```

**NOTE on maintainability:** Another way of solving this particular problem would be to use a completely new custom field. However, *we only use this as a last resort*.  The reason why we prefer to use the `from:` configuration option is because this approach will adapt over time with changes to the AWS service API model, including documentation *about* those fields, whereas completely new custom fields will always need to be hand-rolled and no API documentation will be auto-generated for them.

### Annotating a field as a "Printer Column"

If we want to add one of a Resource's fields to the output of the `kubectl get` command, we can do so by annotating that field's configuration with a `print:` section.  An example of this is in the EC2 controller's [`ElasticIPAddress` Resource](https://github.com/aws-controllers-k8s/ec2-controller/blob/b161bb67b0e5d8b24588676ae29d0f1e587bd42a/generator.yaml#L244), for which we would like to include the `PublicIP` field in the output of `kubectl get`:

```yaml=
resources:
  ...
  ElasticIPAddress:
    ...
    fields:
      ...
>     PublicIp:
>       print:
>         name: PUBLIC-IP
```
Including this in the field's configuration will cause the code generator to produce `kubebuilder` markers in the appropriate place in its generated code, which will result in the field being included in the `kubectl get` output.

**NOTE** this configuration is used to include printer columns in the output at the level of individual fields.  One can also create [additional printer columns](#including-additional-printer-columns) at the level of Resources.

### Controlling field "late initialization"

[Late initialization of a field](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#controller-assigned-defaults-aka-late-initialization) is a Kubernetes Resource Model concept that allows for a nil-valued field to be defaulted to some value *after the resource has been successfully created*.  This is akin to a database table field's "default value".

Late initialized fields are slightly awkward for an ACK controller to handle, primarily because late initialized fields end up being erroneously identified as having different values in the [Delta comparisons](#the-comparison-hook-points).  The desired state of the field is `nil` but the server-side default value of that field is some non-`nil` value.

ACK's code generator can output Go code that handles this server-side defaulting behaviour (we call this "late initialization"). To instruct the code generator to generate late initialization code for a field, use the `resources[$resource].fields[$field].late_initialize` configuration option.

A good example of late initialization can be found in the RDS controller. For `DBInstance` resources, if the user does not specify an availability zone when creating the `DBInstance`, RDS chooses one for the user.  To ensure that the RDS controller understands that the `AvailabilityZone` field is set to a default value after creation, the [following configuration](https://github.com/aws-controllers-k8s/rds-controller/blob/f97b026cdd72e222390f42a18770fb0de49c3b41/generator.yaml#L211) is set in the `generator.yaml`:

```yaml=
resources:
  DBInstance:
    fields:
      AvailabilityZone:
>        late_initialize: {}
```

**NOTE**: the `late_initialize:` configuration option is currently a struct with a couple member fields that are not yet implemented (as of Dec 2022), which is why you need to use the `{}` notation.

### Informing the code generator that a field refers to another Resource

One custom resource can refer to another custom resource using something called Resource References. The Go code that handles the validation and resolution of Resource References is generated by `ack-generate`.

Use the `resources[$resource].fields[$field].references` configuration option to inform the code generator what *kind* of Resource a field references and which *field* within that Resource is the identifier field.

Here is an example from the API Gateway v2 controller that shows an [Integration Resource](https://github.com/aws-controllers-k8s/apigatewayv2-controller/blob/f94a6ba88adda9790a25540729c89a84f7645ccb/generator.yaml#L63) referencing an API and a VPCLink Resource:

```yaml=
resources:
  Integration:
    fields:
      ApiId:
        references:
          resource: API
          path: Status.APIID
      ConnectionId:
        references:
          resource: VPCLink
          path: Status.VPCLinkID
```

After regenerating the controller, the Integration resource will have two *new* fields, one called `APIRef` and another called `ConnectionRef`. The [Go type of these fields](https://github.com/aws-controllers-k8s/apigatewayv2-controller/blob/5e346c359c25cf29be93b3f3bca30c59cc21a9bf/apis/v1alpha1/integration.go#L26-L31) will be a pointer to an `AWSResourceReferenceWrapper`:

```go=
type IntegrationSpec struct {
	APIID  *string                                  `json:"apiID,omitempty"`
	APIRef *ackv1alpha1.AWSResourceReferenceWrapper `json:"apiRef,omitempty"`

	ConnectionID  *string                                  `json:"connectionID,omitempty"`
	ConnectionRef *ackv1alpha1.AWSResourceReferenceWrapper `json:"connectionRef,omitempty"`
    ...
}
```

**NOTE**: The generated name of the reference fields will always be the field name, stripped of any "Id", "Name", or "ARN" suffix, plus "Ref".

In the above example, both the API and VPCLink Resources are managed by the API Gateway v2 controller. It is possible to reference Resources that are managed by a *different* ACK controller by specifying the `resources[$resource].fields[$field].references.service_name` configuration option, as shown in this example from the RDS controller, which has the [DBCluster resource reference the KMS controller's Key resource](https://github.com/aws-controllers-k8s/rds-controller/blob/f8b5d69f822bfc809cbfa25ef7ad60b58a4af22e/generator.yaml#L111-L115):

```yaml=
resources:
  DBCluster:
    fields:
      KmsKeyId:
        references:
          resource: Key
          service_name: kms
          path: Status.ACKResourceMetadata.ARN
```

### Adding custom fields

`resources[$resource].fields[$field].type` *overrides* the inferred Go type of a field. This is required for custom fields that are not inferred (either as a `Create` Input/Output shape or via the `SourceFieldConfig` attribute).

As an example, assume you have a `Role` Resource where you want to add a custom `Spec` field called `Policies` that is a slice of string pointers.

The `generator.yaml` file for the IAM controller [looks like this](https://github.com/aws-controllers-k8s/iam-controller/blob/3f60454e25ce47c050d429773aa826253bb21507/generator.yaml#L172):

```yaml=
resources:
  Role:
    fields:
      Policies:
        type: []*string
```

There is no `Policies` field in either the `CreateRole` Input or Output shapes, therefore in order to create a new custom field, we simply add a `Policies` object in the `fields` configuration mapping and tell the code generator what Go type this new field will have -- in this case, `[]*string`.

**NOTE on maintainability**: Use custom fields as a last resort! When you use custom fields, you will not get the benefit of auto-generated documentation for your field like you will with auto-inferred or `from:`-configured fields. **You will be required to use custom code hooks to populate and set any custom fields**.

**NOTE: (DEPRECATED)** This can also be accomplished by using [`custom_field:`](https://github.com/aws-controllers-k8s/ec2-controller/blob/b161bb67b0e5d8b24588676ae29d0f1e587bd42a/generator.yaml#L361), however we intend to move away from this approach.

[Relevant `TODO` for combining CustomShape stuff into `type:` override](https://github.com/aws-controllers-k8s/ec2-controller/blob/b161bb67b0e5d8b24588676ae29d0f1e587bd42a/generator.yaml#L361)

## Custom code hook points

The code generator will generate Go code that implements the `aws-sdk-go` SDK "binding" calls.  Sometimes you will want to inject bits of custom code at various points in the code generation pipeline.

Custom code [hook points](https://github.com/aws-controllers-k8s/code-generator/blob/160b839fe09dd7e1f321e094604ffc3b6ae2a285/pkg/generate/ack/hook.go#L28) do this injection.  They should be preferred versus using complete overrides (e.g.,  `resources[$resource].update_operation.custom_method_name`).  The reason that custom code hooks are preferred is because you generally want to maximize the amount of *generated* code and minimize the amount of *hand-written* code in each controller.  *[NOTE(jljaco): decide later whether to bother documenting complete overrides via `update_operation.custom_method_name`]*

### The `sdk.go` hook points

First, some background.  Within the `pkg/resources/$resource/sdk.go` file, there are 4 primary resource manager methods that control CRUD operations on a resource:

* `sdkFind` reads a single resource record from a backend AWS service API, then populates a custom resource representation of that record and returns it back to the reconciler.
* `sdkCreate` takes the desired custom resource state (in the `Spec` struct of the CR).  It calls AWS service APIs to create the resource in AWS, then sets certain fields on the custom resource's `Status` struct that represent the latest observed state of that resource.
* `sdkUpdate` takes the desired custom resource state (from the `Spec` struct of the CR), the latest observed resource state, and a representation of the differences between those (called a `Delta` struct). It calls one or more AWS service APIs to modify a resource's attributes, then populates the custom resource's `Status` struct with the latest (post-modification) observed state of the resource.
* `sdkDelete` calls one or more AWS service APIs to delete a resource.

For all 4 of these main ResourceManager methods, there is a consistent code path that looks like this:

1. **Construct the SDK Input shape**. For `sdkFind` and `sdkDelete`, this Input shape will contain the identifier of the resource (e.g. an `ARN`). For `sdkCreate` and `sdkUpdate`, this Input shape will also contain various desired state fields for the resource. This is called the "**Set SDK**" stage and corresponds to code generator functions in code-generator's [`pkg/generate/code/set_sdk.go`](https://github.com/aws-controllers-k8s/code-generator/blob/main/pkg/generate/code/set_sdk.go).
2. **Pass the SDK Input shape** to the `aws-sdk-go` API method.
    - For `sdkFind`, this API method will be *either* the `ReadOne` operation for the resource (e.g., ECR's `GetRepository` or RDS's `DescribeDBInstance`) or the `ReadMany` operation (e.g., S3's `ListBuckets` or EC2's `DescribeInstances`).
    - For the `sdkCreate`, `sdkUpdate` and `sdkDelete` methods, the API operation will correspond to the `Create`, `Update` and `Delete` operation types.
4. **Process the error/return code** from the API call. If there is an error that appears in the list of Terminal codes (**TODO(jljaco) link to docs**), then the custom resource will have a Terminal condition applied to it, and a Terminal error is returned to the reconciler. The reconciler will subsequently add a `ACK.Terminal` Condition to the custom resource.
5. **Process the Output shape** from the API call.  If no error was returned from the API call, the Output shape representing the HTTP response content will then be processed, resulting in fields in either the `Spec` or `Status` of the custom resource being set to the value of matching fields on the Output shape.  This is called the "**Set Resource**" stage and corresponds to code generator functions in code-generator's [`pkg/generate/code/set_resource.go`](https://github.com/aws-controllers-k8s/code-generator/blob/main/pkg/generate/code/set_resource.go).

Along with the above 4 main ResourceManager methods, there are a number of generated helper methods and functions that will:
* create the SDK input shape used when making HTTP requests to AWS APIs
* process responses from those AWS APIs

#### `sdk_*_pre_build_request`

The `sdk_*_pre_build_request` hooks are called _before_ the call to construct the Input shape that is used in the API operation and therefore _before_ any call to validate that Input shape.

Use this custom hook point if you want to short-circuit the processing of the resource for some reason **OR** if you want to process certain resource fields (e.g., Tags) separately from the main resource fields.

##### Example: Short-circuiting

Here is an example from the DynamoDB controller's [`generator.yaml`](https://github.com/aws-controllers-k8s/dynamodb-controller/blob/ce5980c26538b0d9310a2526a845a77da2d2f611/generator.yaml#L1) file that uses a [`pre_build_request`](https://github.com/aws-controllers-k8s/dynamodb-controller/blob/ce5980c26538b0d9310a2526a845a77da2d2f611/generator.yaml#L36) custom code hook for Table resources:

```yaml=
resources:
  Table:
    hooks:
      sdk_delete_pre_build_request:
        template_path: hooks/table/sdk_delete_pre_build_request.go.tpl
```

As you can see, the hook is for the Delete operation.  You can specify the filepath to a template which contains Go code that you wish to inject at this custom hook point. Here is the [Go code from that template](https://github.com/aws-controllers-k8s/dynamodb-controller/blob/1e4563776d5efe9455cb7a347d73cc298f6f16b9/templates/hooks/table/sdk_delete_pre_build_request.go.tpl#L0-L1):

```go=
	if isTableDeleting(r) {
		return nil, requeueWaitWhileDeleting
	}
	if isTableUpdating(r) {
		return nil, requeueWaitWhileUpdating
	}
```

The snippet of Go code above simply requeues the resource to be deleted in the future if the Table is currently either being updated (via [`UpdateTable`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateTable.html)) or deleted (via [`DeleteTable`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_DeleteTable.html)).

After running `make build-controller` for DynamoDB, the above `generator.yaml` configuration and corresponding template file produces the following Go code implementation for `sdkDelete` inside of the `sdk.go` file for Table resources:

```go=
// sdkDelete deletes the supplied resource in the backend AWS service API
func (rm *resourceManager) sdkDelete(
	ctx context.Context,
	r *resource,
) (latest *resource, err error) {
	rlog := ackrtlog.FromContext(ctx)
	exit := rlog.Trace("rm.sdkDelete")
	defer func() {
		exit(err)
	}()
>	if isTableDeleting(r) {
>		return nil, requeueWaitWhileDeleting
>	}
>	if isTableUpdating(r) {
>		return nil, requeueWaitWhileUpdating
>	}
	input, err := rm.newDeleteRequestPayload(r)
	if err != nil {
		return nil, err
	}
	var resp *svcsdk.DeleteTableOutput
	_ = resp
	resp, err = rm.sdkapi.DeleteTableWithContext(ctx, input)
	rm.metrics.RecordAPICall("DELETE", "DeleteTable", err)
	return nil, err
}
```

In the example above, we've highlighted the lines (with `>`) that were injected into the `sdkDelete` method using this custom hook point.

##### Example: Custom field processing

Another example of a `pre_build_request` custom hook comes from the IAM controller's [Role resource](https://github.com/aws-controllers-k8s/iam-controller/blob/3f60454e25ce47c050d429773aa826253bb21507/generator.yaml#L122) and this `generator.yaml` snippet:

```yaml=
resources:
  Role:
    hooks:
      sdk_update_pre_build_request:
        template_path: hooks/role/sdk_update_pre_build_request.go.tpl
```

which has the following [Go code in the template file](https://github.com/aws-controllers-k8s/iam-controller/blob/3f60454e25ce47c050d429773aa826253bb21507/generator.yaml#L122):

```go=
	if delta.DifferentAt("Spec.Policies") {
		err = rm.syncPolicies(ctx, desired, latest)
		if err != nil {
			return nil, err
		}
	}
	if delta.DifferentAt("Spec.Tags") {
		err = rm.syncTags(ctx, desired, latest)
		if err != nil {
			return nil, err
		}
	}
	if delta.DifferentAt("Spec.PermissionsBoundary") {
		err = rm.syncRolePermissionsBoundary(ctx, desired)
		if err != nil {
			return nil, err
		}
	}
	if !delta.DifferentExcept("Spec.Tags", "Spec.Policies", "Spec.PermissionsBoundary") {
		return desired, nil
	}
```

What you can see above is the use of the `pre_build_request` hook point to update the Role's policy collection, tag collection, and permissions boundary _before_ calling the `UpdateRole` API call.  The reason for this is because a Role's policies, tags, and permissions boundary are set using a different set of AWS API calls.

> **TOP TIP (1)**:
> Note the use of `delta.DifferentAt()` in the code above.  This is the recommended best practice for determining whether a particular field at a supplied field path has diverged between the desired and latest observed resource state.

#### `sdk_*_post_build_request`

The `post_build_request` hooks are called AFTER the call to construct the Input shape but _before_ the API operation.

Use this custom hook point if you want to add custom validation of the Input shape.

Here's an example of a [`post_build_request` custom hook point](https://github.com/aws-controllers-k8s/rds-controller/blob/f97b026cdd72e222390f42a18770fb0de49c3b41/generator.yaml#L196) from the RDS controller's DBInstance resource:

```yaml=
resources:
  DBInstance:
    hooks:
      sdk_update_post_build_request:
        template_path: hooks/db_instance/sdk_update_post_build_request.go.tpl
```

and here's the [Go code in that template](https://github.com/aws-controllers-k8s/rds-controller/blob/b0d7dadfce38d293df637b24479ac0a85c764ad9/templates/hooks/db_instance/sdk_update_post_build_request.go.tpl#L0-L1):

```go=
    // ModifyDBInstance call will return ValidationError when the
    // ModifyDBInstanceRequest contains the same DBSubnetGroupName
    // as the DBInstance. So, if there is no delta between
    // desired and latest for Spec.DBSubnetGroupName, exclude it
    // from ModifyDBInstanceRequest
    if !delta.DifferentAt("Spec.DBSubnetGroupName") {
        input.DBSubnetGroupName = nil
    }

    // RDS will not compare diff value and accept any modify db call
    // for below values, MonitoringInterval, CACertificateIdentifier
    // and user master password, NetworkType
    // hence if there is no delta between desired
    // and latest, exclude it from ModifyDBInstanceRequest
    if !delta.DifferentAt("Spec.MonitoringInterval") {
        input.MonitoringInterval = nil
    }
    if !delta.DifferentAt("Spec.CACertificateIdentifier") {
        input.CACertificateIdentifier = nil
    }
    if !delta.DifferentAt("Spec.MasterUserPassword.Name") {
        input.MasterUserPassword = nil
    }
    if !delta.DifferentAt("Spec.NetworkType") {
        input.NetworkType = nil
    }

    // For dbInstance inside dbCluster, it's either aurora or
    // multi-az cluster case, in either case, the below params
    // are not controlled in instance level.
    // hence when DBClusterIdentifier appear, set them to nil
    // Please refer to doc : https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_DeleteDBInstance.html
    if desired.ko.Spec.DBClusterIdentifier != nil {
        input.AllocatedStorage = nil
        input.BackupRetentionPeriod = nil
        input.PreferredBackupWindow = nil
        input.DeletionProtection = nil
    }
```

As you can see, we add some custom validation and normalization of the Input shape for a DBInstance before calling the `ModifyDBInstance` API call.

> **TOP TIP (2)**:
> Note the verbose usage of nil-checks. **_This is very important_**.  `aws-sdk-go` does not have automatic protection against `nil` pointer dereferencing.  *All* fields in an `aws-sdk-go` shape are **pointer types**.  This means you should **always** do your own nil-checks when dereferencing **any** field in **any** shape.

#### `sdk_*_post_request`

The `post_request` hooks are called IMMEDIATELY AFTER the API operation `aws-sdk-go` client call.  These hooks will have access to a Go variable named `resp` that refers to the `aws-sdk-go` client response and a Go variable named `respErr` that refers to any error returned from the `aws-sdk-go` client call.

#### `sdk_*_pre_set_output`

The `pre_set_output` hooks are called BEFORE the code that processes the Outputshape (the `pkg/generate/code.SetOutput` function). These hooks will have access to a Go variable named `ko` that represents the concrete Kubernetes CR object that will be returned from the main method (`sdkFind`, `sdkCreate`, etc). This `ko` variable will have been defined immediately before the `pre_set_output` hooks as a copy of the resource that is supplied to the main method, like so:

```go
	// Merge in the information we read from the API call above to the copy of
	// the original Kubernetes object we passed to the function
	ko := r.ko.DeepCopy()
```

#### `sdk_*_post_set_output`

The `post_set_output` hooks are called AFTER the the information from the API call is merged with the copy of the original Kubernetes object. These hooks will have access to the updated Kubernetes object `ko`, the response of the API call (and the original Kubernetes CR object if it's `sdkUpdate`).

#### `sdk_file_end`

The `sdk_file_end` is a generic hook point that occurs outside the scope of any specific `AWSResourceManager` method and can be used to place commonly-generated code inside the `sdk.go` file.

**_NOTE(jaypipes): This is the weirdest of the hooks... need to cleanly explain this!_**

### The comparison hook points

#### `delta_pre_compare`

The `delta_pre_compare` hooks are called _before_ the generated code that compares two resources.

**NOTE**: If you specified that a particular field should have its **comparison code ignored**, you almost always will want to use a `delta_pre_compare` hook to handle the comparison logic for that field. See the [example above](#delta_pre_compare_example_1) in the section on "*Marking a field as ignored*" for an illustration of this.

The canonical example of when and how to use this custom code hook point is for handling the correct comparison of slices of Tag structs.

By default, if the code generator does not know how to generate specialized comparison code for a Go type, it will generate a call to `reflect.DeepEqual` for this comparison. However, for some types (e.g., lists-of-structs), `reflect.DeepEqual` will return `true` *even when the only difference between two lists lies in the order by which the structs are sorted*. This sort order needs to be ignored in order that the comparison logic properly returns `false` for lists-of-structs that are identical regardless of sort order.

The IAM controller's [`generator.yaml`](https://github.com/aws-controllers-k8s/iam-controller/blob/3f60454e25ce47c050d429773aa826253bb21507/generator.yaml#L124) file contains this snippet:

```yaml
  Role:
    hooks:
      delta_pre_compare:
        code: compareTags(delta, a, b)
    fields:
      Tags:
        compare:
          is_ignored: true
```

and the `delta_pre_compare` hook code is an inline Go code function call to `compareTags`. This function is [defined in the `hooks.go` file](https://github.com/aws-controllers-k8s/iam-controller/blob/b6a62ca48c1aea7ab78e62fb3ad6845335c1f1c2/pkg/resource/role/hooks.go#L188-L202) for the Role resource and looks like this:

```go=
// compareTags is a custom comparison function for comparing lists of Tag
// structs where the order of the structs in the list is not important.
func compareTags(
	delta *ackcompare.Delta,
	a *resource,
	b *resource,
) {
	if len(a.ko.Spec.Tags) != len(b.ko.Spec.Tags) {
		delta.Add("Spec.Tags", a.ko.Spec.Tags, b.ko.Spec.Tags)
	} else if len(a.ko.Spec.Tags) > 0 {
		if !commonutil.EqualTags(a.ko.Spec.Tags, b.ko.Spec.Tags) {
			delta.Add("Spec.Tags", a.ko.Spec.Tags, b.ko.Spec.Tags)
		}
	}
}
```

[`commonutil.EqualTags`](https://github.com/aws-controllers-k8s/iam-controller/blob/b6a62ca48c1aea7ab78e62fb3ad6845335c1f1c2/pkg/util/tags.go#L22-L59) properly handles the comparison of lists of Tag structs.

#### `delta_post_compare`

The `delta_post_compare` hooks are called _after_ the generated code that compares two resources.

This hook is not commonly used, since the `delta_pre_compare` custom code hook point is generally used to inject custom code for comparing special fields.

However, the `delta_post_compare` hook point can be useful if you want to add some code that can post-process the Delta struct *after all fields* have been compared. For example, if you wanted to output some debugging information about the comparison operations.

### The late initialization hook points

#### `late_initialize_pre_read_one`

The `late_initialize_pre_read_one` hooks are called _before_ making the `readOne` call inside the `AWSResourceManager.LateInitialize()` method.
TODO
#### `late_initialize_post_read_one`

The `late_initialize_post_read_one` hooks are called _after_ making the `readOne` call inside the `AWSResourceManager.LateInitialize()` method.
TODO
### The reference hook points

#### `references_pre_resolve`

The `references_pre_resolve` hook is called _before_ resolving the references for all Reference fields inside the `AWSResourceManager.ResolveReferences()` method.
TODO
#### `references_post_resolve`

The `references_post_resolve` hook is called _after_ resolving the references for all Reference fields inside the `AWSResourceManager.ResolveReferences()` method.
TODO
### The tags hook points

#### `ensure_tags`

The `ensure_tags` hook provides a complete custom implementation for the `AWSResourceManager.EnsureTags()` method.
TODO

#### `convert_tags`

The `convert_tags` hook provides a complete custom implementation for the `ToACKTags` and `FromACKTags` methods.
TODO

#### `convert_tags_pre_to_ack_tags`

The `convert_tags_pre_to_ack_tags` hooks are called _before_ converting the K8s resource tags into ACK tags.
TODO

#### `convert_tags_post_to_ack_tags`

The `convert_tags_post_to_ack_tags` hooks are called _after_ converting the K8s resource tags into ACK tags.
TODO

#### `convert_tags_pre_from_ack_tags`

The `convert_tags_pre_from_ack_tags` hooks are called _before_ converting the ACK tags into K8s resource tags.
TODO

#### `convert_tags_post_from_ack_tags`

The `convert_tags_post_from_ack_tags` hooks are called _after_ converting the ACK tags into K8s resource tags.
TODO

## Attribute-based APIs

**OMG TODO.**

## Miscellaneous/maybe cover later/documentation backlog

### What does PrefixConfig do?
### What if the code generator cannot figure out my service's API model name?
### list_operation.match_fields
