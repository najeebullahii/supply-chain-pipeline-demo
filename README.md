# Supply chain security pipeline

A GitHub Actions pipeline that makes sure a container image is scanned, signed and checked before it is deployed to AWS. The app is a tiny FastAPI service. The pipeline is the project.

Think of it like a sealed parcel. The pipeline checks what is inside, puts a tamper seal on it, and the delivery step refuses any parcel whose seal does not check out.

This is a portfolio project in my own AWS account. It is not a production system. The full write-up, including what went wrong, is on Medium: [MEDIUM_LINK](https://medium.com/@ishiyakunajib04/i-built-a-deployment-pipeline-that-refuses-to-run-tampered-code-and-proved-it-1e47803b1c1e)

## What happens on every push to main

1. Trivy scans the source files. The pipeline stops on any fixable HIGH or CRITICAL finding.
2. Docker builds the image.
3. Trivy scans the built image with the same rule.
4. Syft creates a software bill of materials (SBOM) in SPDX JSON, saved as a build artifact.
5. The pipeline signs in to AWS with an OIDC role, so no AWS keys are stored anywhere, and pushes the image to Amazon ECR.
6. Cosign signs the image without a stored key (keyless signing).
7. Cosign verifies the signature and checks that it was made by this workflow on the `main` branch. If this fails, the pipeline stops and nothing is deployed.
8. The pipeline puts the new image into the ECS task definition by digest, not by tag, and deploys it to AWS ECS Fargate. It waits until the service is stable.

## The tamper test

The workflow `tamper-test.yml` is run by hand. It pulls an image that was signed properly, adds a file called `/app/intruder.txt`, pushes it under the tag `tampered-test` and runs the same verification step. The check fails, which is the result I wanted: a changed image cannot get through.

## Results

- The first scan found 14 vulnerabilities.
- The tampered image failed the signature check.
- The AWS cost was $0.00. I left out a load balancer and a NAT gateway to keep it that way.

## Choices I made

- **No stored keys.** The AWS role is limited to one repository, one branch and one ECR repository.
- **Digest, not tag.** A tag can be moved to a different image. A digest cannot.
- **A small image.** It runs as a non-root user, and the build upgrades the operating system packages first.
- **Two ignored findings, with reasons.** `.trivyignore` holds two entries. Each one explains why it is ignored and has a review date of 14 September 2026.

## Limits

- It is one small service, with no load balancer or HTTPS. It is a demonstration, not a production setup.
- The scan finds known vulnerabilities only, and it skips ones that have no fix yet.
- Verification proves who built and signed the image. It does not prove the code is safe.
- It deploys straight from `main` with no approval step. A real team would add one.
- The ECR repository, the ECS cluster, service and task definition, and the IAM role are not defined in this repository.

## Running it yourself

1. Create an ECR repository named `supply-chain-pipeline-demo`.
2. Create an ECS Fargate cluster, service and task definition with the names used in `.github/workflows/ci.yml`.
3. Create an IAM role that trusts GitHub's OIDC provider for your repository and the `main` branch only, with permission to push to ECR and deploy to the ECS service.
4. In both workflow files, replace the role ARN and the region with yours.
5. Push to `main`.

## Files

| File | What it does |
|---|---|
| `.github/workflows/ci.yml` | The pipeline: scan, build, scan, SBOM, push, sign, verify, deploy |
| `.github/workflows/tamper-test.yml` | The manual tamper test |
| `.trivyignore` | The two ignored findings and why |
| `Dockerfile` | The image: slim Python, non-root user |
| `main.py` | The demo app, with `/` and `/health` |
