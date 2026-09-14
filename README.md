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

```sh
export MY_KEY_NAME="studentXX-key"

aws ec2 create-key-pair \
  --key-name "$MY_KEY_NAME" \
  --query "KeyMaterial" \
  --output text > ./"$MY_KEY_NAME".pem

ls -l *.pem
```

```sh
chmod 400 *.pem
ls -l *.pem
```

```sh
# 1. 서울 리전 최신 Ubuntu 26.04 ARM AMI 조회
export AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/26.04/stable/current/arm64/hvm/ebs-gp3/ami-id \
  --query "Parameter.Value" --output text)
echo "최신 Ubuntu ARM AMI:$AMI_ID"

# 2. 인스턴스 프로비저닝
export INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t4g.nano \
  --key-name "$MY_KEY_NAME" \
  --security-group-ids "$MY_SG_ID" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=student01-test-ec2},{Key=Course,Value=infra-training}]" \
  --query "Instances[0].InstanceId" --output text)
echo "프로비저닝 인스턴스 ID:$INSTANCE_ID"

# 3. 상태 대기 (임의의 sleep 대신 공식 waiter 사용)
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"

# 4. 공인 IP 조회
export PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "할당된 퍼블릭 IP:$PUBLIC_IP"
```

```sh
ssh -i "$MY_KEY_NAME".pem -o StrictHostKeyChecking=accept-new ubuntu@"$PUBLIC_IP"

uname -m                                  # 출력: aarch64
grep PRETTY_NAME /etc/os-release          # 출력: Ubuntu 26.04.1 LTS
free -h | head -2                         # t4g.nano 메모리 약 405Mi

exit
```

```sh
aws ec2 stop-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-stopped --instance-ids "$INSTANCE_ID"
echo "인스턴스가 안전하게 중지(Stopped)되었습니다."
```