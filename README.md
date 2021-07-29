# Procedure to set up CI/CD with gitops concepts

## 1. Create two githup repositories; App, K8S Manifests

실습을 위해 두 개의 github 레파지토리가 필요 합니다.

- Application 용 레파지토리: Frontend 소스가 위치한 레파지토리
- K8S Manifests 용 레파지토리: K8S 관련 메니페스트가 위치한 레파지토리

**(1)** **_front-app-repo_** 라는 이름으로 Application repository 생성

![CreateAppRepo](./images/create_app_repo.png)

**(2)** git remote 설정을 위해, amazon-eks-frontend 디렉토리의 git 초기화

```bash
cd ~/environment/amazon-eks-frontend
rm -rf .git
```

**(3)** git remote repo setup
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

> 만약 github MFA 인증을 사용 하고 있는 경우, personal access token을 만들어 password로 사용해야 할 수 있습니다. personal access token을 만드는 방법은 [다음 github 안내](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token) 1-10을 따릅니다. 그리고 얻은 token 값을 push 과정에서 묻는 password 에 대한 응답으로 입력 하면 됩니다.

## 2. Create IAM for CI/CD with least privileges

front app 을 빌드 하고, docker 이미지로 만든 다음 이를 ECR 에 push 하는 과정은 gitHub Action을 통해 이루어 집니다. 이 과정에서 사용할 IAM User를 least privilege 를 준수하여 생성 합니다.

**(1)** IAM user 생성

```bash
aws iam create-user --user-name github-action
```

**(2)** ECR policy 생성

생성할 policy 파일을 만듭니다.

```bash
cd ~/environment
cat <<EOF>> ecr-policy.json
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
            "Resource": "arn:aws:ecr:ap-northeast-2:120340950114:repository/demo-frontend"
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

**(3)** ECR policy를 IAM user에 부여
생성한 ecr-policy를 새로 생성한 IAM user 에게 할당 합니다.

```bash
aws iam attach-user-policy --user-name github-action --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ecr-policy
```

## 3. Set up githup secrets(AWS Credential, githup token)

github action 에서 사용할 AWS credential, github token을 생성하고, 설정 합니다.

**(1)** AWS Credential 생성

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

**(2)** github personal token 생성

github.com 로그인 후 User profile > Settings > Developer settings > Personal access tokens 으로 이동 합니다.
그리고 우측 상단에 위치한 Generate new token을 선택 합니다.

Note 에 `access token for github action` 라 입력 하고 **Select scopes** 에서 **repo** 를 선택 합니다. 그리고 화면 아래에서 **Generate token** 을 클릭 합니다.

![Personal Access Token](./images/personal_access_token.png)

그리고 화면에 출력되는 token 값을 복사 합니다.

![Copy Access Token](./images/copy_token.png)

**(3)** Set up github secret

**front-app-repo** 레파지토리로 돌아가 **Settings > Secrets** 을 선택 합니다. 그리고 이어 화면 우측 상단의 **New repository secret** 을 클릭 합니다.

아래 화면 같이 **Name**에 `ACTION_TOKEN` **Value** 에는 앞서 복사한 personal access token 값을 넣은 후 **Add secret** 을 클릭 합니다.

![Action Token Secret](./images/action_token_secret.png)

다음은 마찬가지 절차로, 앞서 생성 후 기록/저장 해둔 IAM USER 인 **`github-action`** 의 `AccessKeyId` 와 `SecretAccessKey`의 값을 Secret 에 저장 합니다.

이때 `AccessKeyId` 와 `SecretAccessKey`의 각각의 Name 값은 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` 로 합니다.

![Aws Access Key Secret](./images/aws_access_key.png)

![Aws Secret Access Key Secret](./images/aws_secret_access_key.png)

## 4. Create build script for githup action

## 5. Install Kustomize in C9 for local testing

## 6. Install ArgoCD in eks cluster

## 7. Configure ArgoCD in eks cluster
