# App Services Repository
Source code repository for application microservices, including backend/frontend services, Dockerfiles, and CI pipelines.

## ECR deployment

The `ecr-images` GitHub Actions workflow builds the six application images and pushes both the mutable `latest` tag and an immutable `sha-<commit>` tag to the matching ECR repositories. After all images are pushed, it updates `imageRevision` in the GitOps chart. Argo CD syncs that GitOps commit, changing the pod template and causing new pods to pull `latest`.

Configure these repository secrets before running the workflow:

- `AWS_ROLE_ARN`: an IAM role trusted for GitHub Actions OIDC from `MGPZZ/App` on the `main` branch, with push access to the six application ECR repositories.
- `GITOPS_TOKEN`: a GitHub token with contents write access to `MGPZZ/GitOps`, used to update the chart after the image push.

Run the workflow once before applying the Kubernetes Terraform configuration so the six `latest` images exist in ECR. Pushes to `main` that change files under `src/` build and deploy the application images.
