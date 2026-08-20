# TTC README

- [`ubuntu/standard/6.0/Dockerfile`](ubuntu/standard/6.0/Dockerfile) and [`al/aarch64/standard/3.0/Dockerfile`](al/aarch64/standard/3.0/Dockerfile) are used for the web-console-codebuild custom image.
- [`ubuntu/standard/7.0/Dockerfile`](ubuntu/standard/7.0/Dockerfile) is used for the api-codebuild custom image.

## Building and pushing a custom image

1. `cd` into the directory containing the Dockerfile you want to build, e.g. to build the Ubuntu 22.04 (`standard:7.0`) image:

   ```
   cd ubuntu/standard/7.0
   ```

2. Build the image, tagging it as the corresponding AWS CodeBuild curated image name:

   ```
   docker build -t aws/codebuild/standard:7.0 .
   ```

3. Tag the image for your ECR repository:

   ```
   docker tag aws/codebuild/standard:7.0 <account-id>.dkr.ecr.<region>.amazonaws.com/<repository>:7.0
   ```

4. Authenticate Docker to ECR and push the image:

   ```
   aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
   docker push <account-id>.dkr.ecr.<region>.amazonaws.com/<repository>:7.0
   ```

5. In the CodeBuild project settings, set the environment image to the ECR image pushed above, and set **Image pull credentials** (`ImagePullCredentialsType`) to `SERVICE_ROLE` so CodeBuild uses the project's service role (instead of the CodeBuild service principal) to pull the image from ECR.
