```sh
command -v aws
aws --version
# macOS / Linux / WSL2
curl -fsSL https://awscli.amazonaws.com/v2/install.sh | bash\nexport PATH="$HOME/.local/bin:$PATH"
# Windows (Git Bash): 공식 MSI 설치 관리자 내려받아 설치
curl -o AWSCLIV2.msi https://awscli.amazonaws.com/AWSCLIV2.msi
MSYS_NO_PATHCONV=1 msiexec.exe /i AWSCLIV2.msi /qn
aws --version
```

```sh
aws configure sso --profile studentXX

# SSO session name: infra-training
# SSO start URL: https://infra-lab-ai.awsapps.com/start
# SSO region: ap-northeast-2
# SSO registration scopes: sso:account:access
# CLI default client Region: ap-northeast-2
# CLI default output format: json
# CLI profile name: studentXX

aws sts get-caller-identity --profile studentXX
```

```sh
# 세션 갱신 및 환경변수 고정
aws sso login --profile studentXX
export AWS_PROFILE="studentXX"
export AWS_REGION="ap-northeast-2"
export AWS_PAGER=""

# aws sts get-caller-identity
aws ec2 describe-availability-zones \
  --filters "Name=state,Values=available" \
  --query "AvailabilityZones[].{Zone:ZoneName,State:State}" \
  --output table
```

```sh
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" --output text)
echo "기본 VPC ID:$VPC_ID"
```

```sh
# export MY_SG_NAME="student98-web-sg"
export MY_SG_NAME="studentXX-web-sg"
export MY_SG_ID=$(aws ec2 create-security-group \
  --group-name "$MY_SG_NAME" \
  --description "Security Group for Spring Boot and Nginx Practice" \
  --vpc-id "$VPC_ID" \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=$MY_SG_NAME},{Key=Course,Value=infra-training}]" \
  --query "GroupId" --output text)
echo "생성된 보안 그룹 ID:$MY_SG_ID"
```

```sh
# 1. 내 공인 IP 자동 감지
export MY_IP=$(curl -fsS https://checkip.amazonaws.com)
echo "내 공인 IP:$MY_IP"

# 2. SSH는 내 IP만, 웹 포트는 전역 허용
aws ec2 authorize-security-group-ingress --group-id "$MY_SG_ID" --protocol tcp --port 22 --cidr "$MY_IP/32"
aws ec2 authorize-security-group-ingress --group-id "$MY_SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id "$MY_SG_ID" --protocol tcp --port 8080 --cidr 0.0.0.0/0

# 3. 등록 결과 확인
aws ec2 describe-security-groups --group-ids "$MY_SG_ID" \
  --query "SecurityGroups[0].IpPermissions[].{Port:FromPort,Proto:IpProtocol,Cidr:IpRanges[0].CidrIp}" \
  --output table
```
