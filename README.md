# Resolve Job Template

Resolves Kubernetes job or workflow template resource names by label discovery. This action finds the actual resource name in a cluster using label selectors, supporting CronJobs, Argo WorkflowTemplates, and Argo CronWorkflows.

## Usage

```yaml
- name: Resolve job resource
  uses: KoalaOps/resolve-job-template@v1
  id: resolve
  with:
    job_name: my-batch-job
    namespace: production
    job_type: kubernetes-cronjob

- name: Use resolved values
  run: |
    echo "Resource Name: ${{ steps.resolve.outputs.name }}"
    echo "Resource Kind: ${{ steps.resolve.outputs.kind }}"
```

## Inputs

| Input | Description | Required | Valid Values |
|-------|-------------|----------|--------------|
| `job_name` | Logical job name (used for label discovery) | ✅ | Any string |
| `namespace` | Kubernetes namespace where the job is deployed | ✅ | Valid K8s namespace |
| `job_type` | Type of job resource | ✅ | `kubernetes-cronjob`, `argo-workflow`, `argo-cronworkflow` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `name` | Resolved resource name | `my-batch-job-cron` |
| `kind` | Resource kind | `cronjob`, `workflowtemplate`, or `cronworkflow` |

## Job Types

### Kubernetes CronJob
```yaml
- uses: KoalaOps/resolve-job-template@v1
  with:
    job_name: nightly-backup
    namespace: production
    job_type: kubernetes-cronjob
```

### Argo WorkflowTemplate
```yaml
- uses: KoalaOps/resolve-job-template@v1
  with:
    job_name: data-processing
    namespace: workflows
    job_type: argo-workflow
```

### Argo CronWorkflow
```yaml
- uses: KoalaOps/resolve-job-template@v1
  with:
    job_name: scheduled-report
    namespace: workflows
    job_type: argo-cronworkflow
```

## Label Requirements

This action discovers resources using the `app.kubernetes.io/name` label. Your job resources must be labeled:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-actual-cronjob-name
  labels:
    app.kubernetes.io/name: my-batch-job  # This is the job_name input
```

## Error Handling

The action will fail if:
- No resources match the label selector
- Multiple resources match (labels must be unique)
- Invalid job_type is specified
- kubectl is not configured/authenticated

### Common Errors

**No resource found:**
```
::error::No cronjob found in ns=production with label 'app.kubernetes.io/name=my-job'.
Ensure the resource is deployed and labeled.
```

**Multiple resources found:**
```
::error::3 cronjob resources match 'app.kubernetes.io/name=my-job' in ns=production.
Labels must be unique per job. Add app.kubernetes.io/instance or refine labels.
```

## Complete Example

```yaml
name: Execute Job

on:
  workflow_dispatch:
    inputs:
      job_name:
        description: "Job name"
        required: true
      namespace:
        description: "Namespace"
        required: true
      job_type:
        description: "Job type"
        required: true

jobs:
  execute:
    runs-on: ubuntu-latest
    steps:
      - name: Authenticate to cluster
        uses: KoalaOps/cloud-login@v1
        with:
          provider: gcp
          account: my-project
          location: us-central1
          cluster: production

      - name: Resolve job resource
        id: resolve
        uses: KoalaOps/resolve-job-template@v1
        with:
          job_name: ${{ inputs.job_name }}
          namespace: ${{ inputs.namespace }}
          job_type: ${{ inputs.job_type }}

      - name: Create job from CronJob
        if: steps.resolve.outputs.kind == 'cronjob'
        run: |
          kubectl create job \
            "${{ steps.resolve.outputs.name }}-manual-$(date +%s)" \
            --from=cronjob/${{ steps.resolve.outputs.name }} \
            -n ${{ inputs.namespace }}
```

## Notes

- Requires kubectl to be installed and configured
- Cluster authentication must be completed before using this action
- The `app.kubernetes.io/name` label is the primary selector
- For uniqueness in multi-environment setups, consider adding `app.kubernetes.io/instance` labels
- This action only resolves the resource name; it does not execute or modify resources

## License

MIT
