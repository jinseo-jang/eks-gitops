# Procedure to set up CI/CD with gitops concepts

# Introduction

> 이 튜토리얼은 [Amazon EKS로 웹 애플리케이션 구축하기 실습](https://aws-eks-web-application.workshop.aws/ko/)을 바탕으로 gitops pipeline 구성을 안내 합니다. 따라서 해당 실습의 [8.AWS Fargate 사용하기](https://aws-eks-web-application.workshop.aws/ko/80-fargate.html) 까지 완료한 상태에서 출발 합니다. 또한 모든 코드와 튜토리얼은 **Cloud9** 환경을 가정하고 있으므로 로컬PC 환경에서 실행할 경우 **_다랙토리_** 위치에 주의해야 합니다.

이 튜토리얼은 **Github** 에 위치한 application 소스와 k8s manifests 의 변경 사항이 발생될 때 마다 자동으로 **Github Action** 을 통해서 빌드/통합(Continous Integration) 하고, 통합된 배포 대상을 **ArgoCD** 를 통해 k8s 클러스터에 배포(Continous Deployment) 하는 일종의 **gitops** 파이프라인을 만드는 과정을 안내 합니다. 이 때 k8s manifests를 통합/패키징 하는 도구로 **Kustomize**를 사용 합니다.

![gitops](./images/gitops.png)

## Tools to be leveraged

- **GitHub**
- **GitHub Actions**
- **Kustomize**
- **ArgoCD**

# 1. Create two githup repositories; App, K8S Manifests

실습을 위해 두 개의 github 레파지토리가 필요 합니다.

- Application 용 레파지토리: Frontend 소스가 위치한 레파지토리
- K8S Manifests 용 레파지토리: K8S 관련 메니페스트가 위치한 레파지토리

## **(1)** **_front-app-repo_** 라는 이름으로 Application repository 생성

![CreateAppRepo](./images/create_app_repo.png)

## **(2)** git remote 설정을 위해, amazon-eks-frontend 디렉토리의 git 초기화

```bash
cd ~/environment/amazon-eks-frontend
rm -rf .git
```

## **(3)** git remote repo setup

**_front-app-repo_** 에 frontend 소스 코드를 push 합니다.

```bash
cd ~/environment/amazon-eks-frontend
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/jinseo-jang/front-app-repo.git
git push -u origin main
```

이 과정이 완료되면 github에서 확인 합니다.
![AppRepo](./images/result_app_repo.png)

> 만약 push 과정에서, username, password 를 매번 넣어야 하는 상황이 번거롭다면 아래와 같이 cache 설정을 통해 지정 된 시간(기본 15분) 동안 cache 기반으로 로그인 가능 합니다.

```bash
git config --global user.name USERNAME
git config --global user.email EMAIL
git config credential.helper store
git config --global credential.helper 'cache --timeout TIME YOU WANT'
```

> 만약 github MFA 인증을 사용 하고 있는 경우, personal access token을 만들어 password로 사용해야 할 수 있습니다. personal access token을 만드는 방법은 [다음 github 안내](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token) 1-10을 따릅니다.(\*이 때 **Select scopes** 에서 **workflow**를 추가로 선택 합니다) 그리고 얻은 token 값을 push 과정에서 묻는 password 에 대한 응답으로 입력 하면 됩니다.

# 2. Create IAM for CI/CD with least privileges

front app 을 빌드 하고, docker 이미지로 만든 다음 이를 ECR 에 push 하는 과정은 gitHub Action을 통해 이루어 집니다. 이 과정에서 사용할 IAM User를 least privilege 를 준수하여 생성 합니다.

## **(1)** IAM user 생성

```bash
aws iam create-user --user-name github-action
```

## **(2)** ECR policy 생성

생성할 policy 파일을 만듭니다.

```bash
cd ~/environment
cat <<EOF> ecr-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowPush",
            "Effect": "Allow",
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "arn:aws:ecr:ap-northeast-2:${ACCOUNT_ID}:repository/demo-frontend"
        },
        {
            "Sid": "GetAuthorizationToken",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        }
    ]
}
EOF
```

만들어진 파일을 통해 IAM policy를 생성 합니다. 이때 policy 이름으로 `erc-policy` 를 사용 합니다.

```bash
aws iam create-policy --policy-name ecr-policy --policy-document file://ecr-policy.json
```

## **(3)** ECR policy를 IAM user에 부여

생성한 ecr-policy를 새로 생성한 IAM user 에게 할당 합니다.

```bash
aws iam attach-user-policy --user-name github-action --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ecr-policy
```

# 3. Set up githup secrets(AWS Credential, githup token)

github action 에서 사용할 AWS credential, github token을 생성하고, 설정 합니다.

## **(1)** AWS Credential 생성

github action이 빌드된 front app 을 docker image 로 만들어 ECR로 push 합니다. 이 과정에서 AWS credential 을 사용 합니다. 이를 위해 앞서 `github-action`이라는 별도의 least privilege 를 갖는 IAM User를 생성 했습니다. 이제 이 User의 Access Key, Secret Key를 생성 합니다.

```bash
aws iam create-access-key --user-name github-action
```

아래와 같은 출력 결과 가운데 `"SecretAccessKey"`, `"AccessKeyId"`값을 따로 메모 저장 합니다. 이 값은 향후에 사용 합니다.

```json
{
  "AccessKey": {
    "UserName": "github-action",
    "Status": "Active",
    "CreateDate": "2021-07-29T08:41:04Z",
    "SecretAccessKey": "ccccccGCUIFGW45sktvHdJfjSiXa37wJ6ysn+5cX",
    "AccessKeyId": "AKIA2BVCU3P4OFQAZXSW"
  }
}
```

## **(2)** github personal token 생성

github.com 로그인 후 User profile > Settings > Developer settings > Personal access tokens 으로 이동 합니다.
그리고 우측 상단에 위치한 Generate new token을 선택 합니다.

Note 에 `access token for github action` 라 입력 하고 **Select scopes** 에서 **repo** 를 선택 합니다. 그리고 화면 아래에서 **Generate token** 을 클릭 합니다.

![Personal Access Token](./images/personal_access_token.png)

그리고 화면에 출력되는 token 값을 복사 합니다.

![Copy Access Token](./images/copy_token.png)

## **(3)** Set up github secret

**front-app-repo** 레파지토리로 돌아가 **Settings > Secrets** 을 선택 합니다. 그리고 이어 화면 우측 상단의 **New repository secret** 을 클릭 합니다.

아래 화면 같이 **Name**에 `ACTION_TOKEN` **Value** 에는 앞서 복사한 personal access token 값을 넣은 후 **Add secret** 을 클릭 합니다.

![Action Token Secret](./images/action_token_secret.png)

다음은 마찬가지 절차로, 앞서 생성 후 기록/저장 해둔 IAM USER 인 **`github-action`** 의 `AccessKeyId` 와 `SecretAccessKey`의 값을 Secret 에 저장 합니다.

이때 `AccessKeyId` 와 `SecretAccessKey`의 각각의 Name 값은 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` 로 합니다.

![Aws Access Key Secret](./images/aws_access_key.png)

![Aws Secret Access Key Secret](./images/aws_secret_access_key.png)

# 4. Create build script for githup action

## **(1)** create .github folder for github action

```bash
cd ~/environment/amazon-eks-frontend
mkdir -p ./.github/workflows
```

## **(2)** Crate build.yaml to be used to execute github action job

front app 을 checkout 하고, build 한 다음, docker container 로 만들어 ECR 로 push 하는 과정을 담고 있는 github action build 스크립트를 작성 합니다.

이 build 스크립트 내용 중 주목할 부분은 **docker image tag 입니다. `$IMAGE_TAG` 값을 빌드 마다 랜덤한 값으로 만들어 이미지에 부착하여 ECR로 push 합니다.**

```bash
cd ~/environment/amazon-eks-frontend/.github/workflows
cat > build.yaml <<EOF

name: Build Front

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v2

      - name: Check Node v
        run: node -v

      - name: Build front
        run: |
          npm install
          npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: \${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: \${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: $AWS_REGION

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Get image tag(version)
        id: image
        run: |
          VERSION=\$(echo \${{ github.sha }} | cut -c1-8)
          echo VERSION=\$VERSION
          echo "::set-output name=version::\$VERSION"

      - name: Build, tag, and push image to Amazon ECR
        id: image-info
        env:
          ECR_REGISTRY: \${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: demo-frontend
          IMAGE_TAG: \${{ steps.image.outputs.version }}
        run: |
          echo "::set-output name=ecr_repository::\$ECR_REPOSITORY"
          echo "::set-output name=image_tag::\$IMAGE_TAG"
          docker build -t \$ECR_REGISTRY/\$ECR_REPOSITORY:\$IMAGE_TAG .
          docker push \$ECR_REGISTRY/\$ECR_REPOSITORY:\$IMAGE_TAG

EOF
```

## **(3)** Push code to front-app-repo

이제 코드를 front-app-repo 로 push 하여 github action workflow를 동작 시킵니다. 위에서 작성한 build.yaml 을 기반으로 github action이 동작 합니다.

```bash
cd ~/environment/amazon-eks-frontend
git add .
git commit -m "Add github action build script"
git push origin main
```

github 화면으로 돌아가 변경 사항이 push 되었는지 확인하고, github action workflow가 다음과 같이 동작 하는지 확인 합니다.

![Github Action Start](./images/githubaction_start.png)
![Github Action Detail](./images/githubaction_detail.png)

정상적인 build가 확인 되었다면 front app의 이미지가 사용하는 ECR 레파지토리인 **`demo-fronted`** 로 돌아가, 새로운 `$IMAGE_TAG`를 갖는 이미지가 push 되었는지 확인 합니다.

sha 값의 일부가 포함된 Image Tag로 push 된 이미지가 확인 됩니다.

![New Image Tag Pushed](./images/new_image_tag.png)

# 5. Kustomize Overview

Kustomize 는 쿠버네티스 manifest 를 사용자 입맛에 맞도록 정의 하는데 도움을 주는 도구 입니다. 여기서 "사용자 입맛에 맞도록 정의"에 포함 되는 경우는, 모든 리소스에 동일한 네임스페이스를 정의 하거나, 동일한 네임 접두사/접미사를 준다거나, 동일한 label 을 주거나, 이미지 태그 값을 변경 하는 것들이 있을 수 있습니다.

본 실습 에서는 Kustomize 를 활용해 kuberenetes Deployment 리소스에 동일한 label, metadata 값 을 주도록 하며, front app의 새로운 변경 사항 발생에 따른 새로운 Image Tag를 Deployment 리소스에 적용할 겁니다.

Kustomize 에 관한 자세한 내용은 [다음](https://kustomize.io/)을 참고 하세요.

# 6. Structure Kubernetes manifest directory to be used with Kustomize

## **(1)** Create directories

Kubernetes manifest 들은 이제 별도의 독립된 github repository를 갖게 될겁니다. 그리고 Kustomize를 통해 manifest 들을 구성 할 겁니다. 따라서 이를 위해 디렉토리 구조를 만듭니다.

```bash
cd ~/environment
mkdir -p ./k8s-manifest-repo/base
mkdir -p ./k8s-manifest-repo/overlays/dev
cd ~/environment/manifests
cp *.yaml ../k8s-manifest-repo/base
cd ../k8s-manifest-repo/base
ls -rlt
```

결과적으로 만들어지는 디렉토리 구조는 다음과 같습니다. _`k8s-manifest-repo`_ 디렉토리 아래에 _`base`_, _`overlays/dev`_ 구조가 생깁니다.

- _`base`_ : kubernetes manifest 원본이 위치한 디렉토리 입니다. 이 안에 위치한 manifest 들은 _`overlays`_ 아래에 위치한 **kustomize.yaml** 파일에 담긴 **사용자 지정 설정** 내용에 따라 변경 변경 되어 집니다.
- _`overlays`_ : **사용자 입맛에 맞는** 설정 값이 위치한 디렉토리 입니다. 이 설정은 **kustomize.yaml** 에 담습니다. 이 하위에 있는 _`dev`_ 디렉토리는 실습을 위해 만든 것으로, 개발 환경에 적용할 설정 파일을 모아 두기 위함 입니다.

**(2)** create Kustomize manifests

이번 실습은 frontend app에 대한 배포 체계를 만드는데 목표로 합니다. 따라서 frontend-deployment.yaml 과 frontend-service.yaml 파일을 kustomize 를 통해 배포 시점에 의도한 값(e.g. Image Tag)을 반영 할겁니다. 아래는 반영될 값 입니다.

- **`metadata.labels`:** `"env: dev"`을 frontenc-deployment.yaml, frontend-service.yaml 에 일괄 반영 합니다.
- **`spec.selector`** : `"select.app: frontend-fargate"` 를 frontend-deployment.yaml, frontend-service.yaml 에 일괄 반영 합니다.
- **`spec.template.spec.containers.image`** : `"image: "` 값을 새롭게 변경된 Image Tag 정보로 업데이트 합니다.

![Kustomize Dir](./images/kustomize_dir.png)

아래와 같이 kustomize.yaml 파일을 만듭니다. 이 파일은 kustomize을 통해 관리/변경 할 kubernetes manifest 대상을 정의 하는 목적 입니다.

```bash
cd ~/environment/k8s-manifest-repo/base
cat <<EOF> kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - frontend-deployment.yaml
  - frontend-service.yaml
EOF
```

이제 위에서 정의한 kubernetes manifest 대상에 어떤 값들을 입맛에 맞게 적용할지 를 결정 하기 위한 파일을 만듭니다. 먼저 fronted-deployment.yaml 을 위한 설정 값을 정의 하겠습니다.

```bash
cd ~/environment/k8s-manifest-repo/overlays/dev
cat <<EOF> front-deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-frontend
  namespace: default
  labels:
    env: dev
spec:
  selector:
    matchLabels:
      app: frontend-fargate
  template:
    metadata:
      labels:
        app: frontend-fargate
EOF
```

다음은 frontend-service.yaml 을 위한 설정 값을 정의 하겠습니다.

```bash
cd ~/environment/k8s-manifest-repo/overlays/dev
cat <<EOF> front-service-patch.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-frontend
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/"
  labels:
    env: dev
spec:
  selector:
    app: frontend-fargate
EOF
```

마지막으로 위에서 설정 한 파일들(값)을 사용하고 frontend app 빌드에 따라 만들어진 새로운 **Image Tag** 를 사용 하겠다고 정의 하겠습니다. 구체적으로는, `name` 에 지정된 image는 `newName`의 image와 `newTag`의 값으로 사용 하겠다는 의미 입니다.

이를 활용해 `newTag` 값을 변경해 새로운 배포가 이루어질 때 마다 이를 kubernetes 클러스터에까지 변경 할 수 있습니다.

아래 명령을 실행 합니다.

```bash
cd ~/environment/k8s-manifest-repo/overlays/dev
cat <<EOF> kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
images:
- name: ${ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/demo-frontend
  newName: ${ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/demo-frontend
  newTag: abcdefg
resources:
- ../../base
patchesStrategicMerge:
- front-deployment-patch.yaml
- front-service-patch.yaml
EOF
```

이상 -patch.yaml 파일에 정의한 내용들은 배포 과정에서 kustomize 에 의해 자동으로 kubernetes manifest 에 반영 됩니다.

# 6. Create github repository for kubernetes manifests

## **(1)** Create github repository for kubernetes manifests

**k8s-manifest-repo** 라는 이름의 github repository를 생성 합니다. 앞서 만든 kubernetes manifests 들이 위치할 공간 입니다.

![Create k8s Repo](./images/create_k8s_repo.png)

앞서 생성한 kubernetes manifests를 새로 생성한 `k8s-manifest-repo`에 push 합니다.

```bash
cd ~/environment/k8s-manifest-repo/
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/jinseo-jang/k8s-manifest-repo.git
git push -u origin main
```

# 7. Install ArgoCD in eks cluster

## **(1)** Install ArgoCD in eks cluster

다음을 실행 하여 ArgoCD를 eks cluster 에 설치 합니다.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

본 실습에서 사용하지는 않지만, ArgoCD 는 CLI을 제공 합니다. 아래를 실행하여 ArgoCD CLI 를 설치 합니다.

```bash
cd ~/environment
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64

sudo chmod +x /usr/local/bin/argocd
```

ArgoCD 서버는 기본적으로 퍼블릭 하게 노출되지 않습니다. 실습의 목적상 이를 변경하여 ELB 를 통해 접속 가능하도록 하겠습니다.

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

3-4 분 정도 후 아래 명령을 통해 ArgoCD 접속이 가능한 ELB 주소를 얻습니다.

```bash
export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output .status.loadBalancer.ingress[0].hostname`
echo $ARGOCD_SERVER
```

ArgoCD의 Username은 `admin` 입니다. 이것이 사용하는 password는 아래 실행을 통해 얻습니다.

```bash
ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
echo $ARGO_PWD
```

위에서 얻은 `$ARGOCD_SERVER`를 브라우저에서 오픈 합니다. 그리고 Username `admin` 을 입력하고 Password 는 `$ARGO_PWD` 값을 입력 합니다.

![ArgoCD Login](./images/argocd_login.png)

# 8. Configure ArgoCD

## **(1)** Configure ArgoCD

로그인 이후 좌측 메뉴에서 애플리케이션 설정 메뉴를 클릭 합니다.

![ArgoCD Menu](./images/argocd_click.png)

그 다음 배포대상이 되는 애플리케이션에 대한 기본 정보를 아래와 같이 입력 합니다.
**Application Name** 은 `eksworkshop-cd-pipeline`, **Project**는 `default`를 입력 합니다.

![ArgoCD General](./images/argocd_general.png)

**SOURCE** 섹션의 **Repository URL** 에는 앞서 생성한 **`k8s-manifest-repo`의 git 주소**, **Revision** 에는 `main`, **Path** 에는 `overlays/dev`를 입력 합니다.

![ArgoCD Source](./images/argocd_source.png)

**DESTINATION** 섹션의 **Cluster URL**에는 `https://kubernetes.default.svc`, **Namespace** 에는 `default`를 입력 하고 상단의 **Create** 를 클릭 합니다.
![ArgoCD Destination](./images/argocd_destination.png)

정상적으로 설정이 마무리 되면 다음과 같이 **eksworkshop-cd-pipeline** 이 생성 됩니다.

![ArgoCD Pipeline](./images/argocd_pipeline.png)

# 9. Add Kustomize build step to githup action build script

## **(1)** Modify gihub action build script

앞서 생성한 github action build 스크립트에 kustomize를 이용하여 컨테이너 image tag 정보를 업데이트 한 후 `k8s-manifest-repo`에 commit/push 하는 단계를 추가 해야 합니다.

추가된 단계가 정상적으로 동작 하면, **ArgoCD**가 `k8s-manifest-repo`를 지켜 보고 있다가 새로운 변경 사항이 발생 되었음을 알아채고, **kustomize build** 작업을 수행하여 새로운 **kubernetes manifest** (\*새로운 image tag를 포함한)를 eks 클러스터에 배포 합니다.

```bash
cd ~/environment/amazon-eks-frontend/.github/workflows
cat <<EOF>> build.yaml

      - name: Setup Kustomize
        uses: imranismail/setup-kustomize@v1

      - name: Checkout kustomize repository
        uses: actions/checkout@v2
        with:
          repository: jinseo-jang/k8s-manifest-repo
          ref: main
          token: \${{ secrets.ACTION_TOKEN }}
          path: k8s-manifest-repo

      - name: Update Kubernetes resources
        run: |
          echo \${{ steps.login-ecr.outputs.registry }}
          echo \${{ steps.image-info.outputs.ecr_repository }}
          echo \${{ steps.image-info.outputs.image_tag }}
          cd k8s-manifest-repo/overlays/dev/
          kustomize edit set image \${{ steps.login-ecr.outputs.registry}}/\${{ steps.image-info.outputs.ecr_repository }}=\${{ steps.login-ecr.outputs.registry}}/\${{ steps.image-info.outputs.ecr_repository }}:\${{ steps.image-info.outputs.image_tag }}
          cat kustomization.yaml

      - name: Commit files
        run: |
          cd k8s-manifest-repo
          git config --global user.email "github-actions@github.com"
          git config --global user.name "github-actions"
          git commit -am "Update image tag"
          git push -u origin main

EOF
```

## **(2)** commit&push new code to front-app-repo

이제 새로운 `build.yaml`을 `front-app-repo`로 push 하여 github action job 을 실행 합니다.

```bash
cd ~/environment/amazon-eks-frontend
git add .
git commit -m "Add kustomize image edit"
git push -u origin main
```

## **(3)** Check github action

github action 화면으로 돌아가 job 이 정상 수행 되는지 확인 합니다.

![Github Action Build](./images/githubaction_build.png)

## **(4)** Check k8s-manifest-repo

`k8s-manifest-repo` 의 commit 상태를 확인 합니다. 아래와 같이 **github-actions** 에 의한 commit 이 확인 됩니다.

![Check Commit](./images/check_commit.png)

## **(5)** Check ArgoCD

ArgoCD 화면으로 돌아가 배포 상태를 확인 합니다. **Applications > eksworkshop-cd-pipeline** 으로 이동 하여 확인 해보면 **CURRENT SYNC STATUS**의 값이 **Out of Synced** 입니다.

git repository 가 변경되면 자동으로 sync 작업이 수행 하도록 하려면 **Auto-Sync** 를 활성화 해야 합니다. 이를 위해 **APP DETAILS** 로 이동 하여 **ENABLE AUTO-SYNC** 버튼을 눌러 활성화 합니다.

![Auto Sync](./images/auto_sync.png)

그러면 `k8s-manifest-repo`의 commit 내용이 ArgoCD에 의해 eks 클러스터에 반영됩니다.

![Auto Sync Run](./images/auto_sync_run.png)

새로운 Image Tag가 정상 반영 되었는지 살펴 보겠습니다.
`k8s-manifest-repo`의 commit history를 통해 변경된 Image Tag 정보를 확인 합니다.

그리고 이 값이 ArgoCD에 의해 새롭게 배포된 `frontend-deployment`가 사용하는 Image Tag 값과 같은지 확인 합니다.

> ArgoCD 메뉴에서 **Applications > eksworkshop-cd-pipeline >** 이동 하여 다이어그램에서 `demo-frontend-`로 시작하는 **pod**을 클릭하면 상세 정보 확인이 가능합니다.)

![Auto Sync Run](./images/git_image_tag.png)

![Auto Sync Run](./images/dp_image_tag.png)

앞으로 `k8s-manifest-repo`의 commit 이 발생할때 마다 ArgoCD가 이를 eks 클러스터에 배포할 것입니다.

# 10. Change frontend application

실제로 frontend application 코드를 변경하여 앞서 만든 gitops pipeline 이 정상적으로 구동되는지 확인 합니다.

## **(1)** Modify frontend application source code

**Cloud9** 으로 이동해, 좌측 폴더 구조에서 **amazon-eks-frontend/src/** 로 이동하여 **`App.js`** 더블 클릭하여 파일을 오픈 합니다.

**line 67**의 값을 **`EKS DEMO Blog version 1`** 으로 변경 하고 저장 합니다. 저장은 **ctrl+s** 를 누릅니다.

```javascript
  return (
    <div className={classes.root}>
      <AppBar position="static" style={{ background: '#2E3B55' }}>
        <Toolbar>
          <IconButton edge="start" className={classes.menuButton} color="inherit" aria-label="menu">
            <CloudIcon />
          </IconButton>
          <Typography
            variant="h6"
            align="center"
            className={classes.title}
          >
            EKS DEMO Blog version 1
          </Typography>
          {new Date().toLocaleTimeString()}
        </Toolbar>
      </AppBar>
      <br/>
```

## **(2)** commit&push

변경 코드를 git repository 에 commit&push 합니다.

```bash
cd ~/environment/amazon-eks-frontend
git add .
git commit -m "Add new blog version"
git push -u origin main
```

## **(3)** Check gitops pipeline and frontend application

소스 commit/push 이후 **github action(build) > ArgoCD(deploy)** 작업이 순서대로 동작 합니다.

아래와 같은 ArgoCD 에 의한 sync 작업이 모두 완료될 때 까지 기다립니다.

![ArgoCD New Deployment](./images/argocd_new_blog_deploy.png)

모두 완료 되면 아래 명령을 통해 얻은 샘플 애플리케이션 주소를 브라우저를 통해 접속 합니다.

```bash
echo http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
```

새로운 변경 사항이 반영된 블로그가 확인 됩니다.

![New Blog Version](./images/new_blog_version.png)

[Lab 1 이어 하기](Lab1.md)
