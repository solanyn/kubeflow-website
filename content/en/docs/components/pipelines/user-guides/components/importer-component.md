+++
title = "Special Case: Importer Components"
description = "Import artifacts from outside your pipeline"
weight = 5
+++

{{% kfp-v2-keywords %}}

### Importer Component Usage

Unlike the other three authoring approaches, an importer component is not a general authoring style but a pre-baked component for a specific use case: loading a machine learning artifact from from a URI into the current pipeline and, as a result, into [ML Metadata][ml-metadata]. This section assumes basic familiarity with KFP [artifacts][artifacts].

As described in [Pipeline Basics][pipeline-basics], inputs to a task are typically outputs of an upstream task. When this is the case, artifacts are easily accessed on the upstream task using `my_task.outputs['<output-key>']`. The artifact is also registered in ML Metadata when it is created by the upstream task.

If you wish to use an existing artifact that was not generated by a task in the current pipeline, you can use a [`dsl.importer`][dsl-importer] component to load the artifact from its URI.

You do not need to write an importer component; it can be imported from the `dsl` module and used directly:

```python
from kfp import dsl

@dsl.pipeline
def my_pipeline():
    task = get_date_string()
    importer_task = dsl.importer(
        artifact_uri='gs://ml-pipeline-playground/shakespeare1.txt',
        artifact_class=dsl.Dataset,
        reimport=True,
        metadata={'date': task.output})
    other_component(dataset=importer_task.output)
```

In addition to an `artifact_uri` argument, you must provide an `artifact_class` argument to specify the type of the artifact.

### Importing Model Artifacts from Container Images

Starting in Kubeflow Pipelines 2.5, you can import model artifacts that are packaged as container images using the [modelcar][model-car] format. Here's how it works:

1. **Specifying the URI**:

   - Use an OCI URI in the `artifact_uri` argument
   - Example: For a container image `quay.io/my-org/my-model:v1`, use `artifact_uri='oci://quay.io/my-org/my-model:v1'`

2. **Runtime Behavior**:

   - When a component uses the imported model, the modelcar runs as a sidecar container in the pod
   - The model's `path` property points to the `/models` directory in the running modelcar

3. **Handling Same User Requirements**:

   - The component container and modelcar container must run with the same user/UID
   - If the users/UIDs don't match, accessing the model's `path` property will fail with a permission error
   - No action is needed for Kubernetes distributions with random user assignment to pods (such as OpenShift, for example)
   - For other cases, you have two options:
     - Build the modelcar container with the same UID as your component container image
     - Set the `PIPELINE_RUN_AS_USER` environment variable to a nonroot UID on the Kubeflow Pipelines API server deployment, which will ensure consistent UIDs on all pods created by Kubeflow Pipelines

### Setting Metadata and Controlling Imports

The `importer` component permits setting artifact metadata via the `metadata` argument. Metadata can be constructed with outputs from upstream tasks, as is done for the `'date'` value in the example pipeline.

You may also specify a boolean `reimport` argument. If `reimport` is `False`, KFP will check to see if the artifact has already been imported to ML Metadata and, if so, use it. This is useful for avoiding duplicative artifact entries in ML Metadata when multiple pipeline runs import the same artifact. If `reimport` is `True`, KFP will reimport the artifact as a new artifact in ML Metadata regardless of whether it was previously imported.

[pipeline-basics]: /docs/components/pipelines/user-guides/components/compose-components-into-pipelines
[dsl-importer]: https://kubeflow-pipelines.readthedocs.io/en/latest/source/dsl.html#kfp.dsl.importer
[artifacts]: /docs/components/pipelines/user-guides/data-handling/artifacts
[ml-metadata]: https://github.com/google/ml-metadata
[model-car]: https://kserve.github.io/website/latest/modelserving/storage/oci/