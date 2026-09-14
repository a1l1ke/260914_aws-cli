```sh
# export MY_KEY_NAME="student98-key"
export MY_KEY_NAME="studentXX-key"
# export AWS_PROFILE="student98"
export AWS_PROFILE="studentXX"
export AWS_REGION="ap-northeast-2"
export AWS_PAGER=""
```

```sh
# 1. 중지된 인스턴스 ID 조회
export INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=stopped" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
echo "대상 인스턴스 ID:$INSTANCE_ID"
# aws ec2 describe-instances
# aws ec2 describe-instances \
#  --filters "Name=instance-state-name,Values=stopped" \
#  --query "Reservations[0].Instances[0].InstanceId"

# 이 과정을 통해서 중지한 인스턴스를 확인
#aws ec2 stop-instances --instance-ids "$INSTANCE_ID"
#aws ec2 wait instance-stopped --instance-ids "$INSTANCE_ID"
#echo "인스턴스가 안전하게 중지(Stopped)되었습니다."

# 위의 describe... 결과가 없다면
# aws ec2 describe-instances \
#  --query "Reservations[0].Instances[0].InstanceId"
```

```sh
# 2. 사양 변경 (t4g.nano -> t4g.micro)
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --instance-type "Value=t4g.micro"

# 3. 시작 및 상태 검사 통과 대기
aws ec2 start-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"
```

```sh
# 4. 새로 할당된 공인 IP 조회
export PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "새로 할당된 공인 IP:$PUBLIC_IP"
```

```sh
# 5. pem 키 권한 설정 후 SSH 접속
chmod 400 "$MY_KEY_NAME".pem
ssh -i "$MY_KEY_NAME".pem -o StrictHostKeyChecking=accept-new ubuntu@"$PUBLIC_IP"
```