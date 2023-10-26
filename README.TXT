// --------------------------------------------------------------------------------
// BUILD IMAGE
// --------------------------------------------------------------------------------

mvn compile dependency:copy-dependencies -DincludeScope=runtime

docker build -t docker-image:test .

// NOTE: If you build this in an M1 Mac, the platform will be arm64, so if you deploy
//       to lambda using x86_64 it will not work

// --------------------------------------------------------------------------------
// TEST
// --------------------------------------------------------------------------------

docker run -p 9000:8080 docker-image:test

curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'

// SHOULD RETURN:
"200 OK"

// --------------------------------------------------------------------------------
// ECR REPO (Do this with CDK later if possible)
// --------------------------------------------------------------------------------

aws ecr create-repository \
    --profile ci_cd \
    --repository-name ci_cd_test --image-scanning-configuration scanOnPush=true --image-tag-mutability MUTABLE

aws ecr set-repository-policy \
    --profile cdk-user --region us-west-2 \
    --repository-name ci_cd_test \
    --policy-text  file://RepositoryPolicy.json

// See RepositoryPolicy.json at the end

PROFILE=github
aws ecr get-login-password \
    --profile $PROFILE \
    | docker login --username AWS --password-stdin 679514643403.dkr.ecr.us-west-2.amazonaws.com

REPO=github_maven_ecr
docker tag docker-image:test 679514643403.dkr.ecr.us-west-2.amazonaws.com/${REPO}:latest

docker push 679514643403.dkr.ecr.us-west-2.amazonaws.com/${REPO}:latest

// --------------------------------------------------------------------------------
// Creating Lambda
// --------------------------------------------------------------------------------

NOTE:   --architectures arm64 is needed if the image is built on a Mac M1

aws lambda create-function \
    --profile cdk-user --region us-west-2 \
    --function-name hello-world \
    --package-type Image \
    --code ImageUri=679514643403.dkr.ecr.us-west-2.amazonaws.com/ci_cd_demo:latest \
    --role arn:aws:iam::679514643403:role/TEST_DELETE_ME \
    --architectures arm64

// --------------------------------------------------------------------------------
// RepositoryPolicy.json
// --------------------------------------------------------------------------------

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::679514643403:user/ci_cd"
      },
      "Action": [
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:GetDownloadUrlForLayer",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ]
    }
  ]
}

// --------------------------------------------------------------------------------
// Clean local docker repo
// --------------------------------------------------------------------------------

for i in `docker image ls | awk -F' ' ' { print $3 }' | grep -v IMAGE`
do
docker image rm $i
done

// --------------------------------------------------------------------------------
// Reset/Re-push git repo just for testing
// --------------------------------------------------------------------------------

rm -rf .git
git init
git status
git add *
git commit -m "Initial test commit"
git remote add main https://git-codecommit.us-west-2.amazonaws.com/v1/repos/ci_cd_demo
git push --set-upstream main main
