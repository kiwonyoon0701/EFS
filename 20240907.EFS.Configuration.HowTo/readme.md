# EFS 생성을 위한 Script 정리

### 다음의 변수들을 수정 필요

```
VPC_ID="vpc-0e7536971adbad31f"  # 사용할 VPC의 ID
EFS_NAME="efs-us-east-1-web2"   # 생성할 EFS의 이름
REGION="us-east-1"              # 사용할 AWS 리전 (미국 동부)
```



---

### 전체적인 Script

::::expand{header="화면 보기" defaultExpanded=false}



```
#!/bin/bash

# 변수 설정
VPC_ID="vpc-0e7536971adbad31f"  # 사용할 VPC의 ID
EFS_NAME="efs-us-east-1-web2"   # 생성할 EFS의 이름
REGION="us-east-1"              # 사용할 AWS 리전 (미국 동부)

# 보안 그룹 ID 가져오기
# 주어진 보안 그룹 이름을 사용하여 보안 그룹 ID를 조회
SECURITY_GROUP_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=web-target-security-group" --query "SecurityGroups[0].GroupId" --output text --region $REGION)

# 보안 그룹 ID가 없으면 오류 메시지 출력 후 종료
if [ -z "$SECURITY_GROUP_ID" ]; then
  echo "보안 그룹 ID를 찾을 수 없습니다!"
  exit 1
fi

echo "Security Group ID: $SECURITY_GROUP_ID"

# 1. Private Subnet ID 가져오기
# VPC 내의 모든 Private Subnet ID를 가져옴
PRIVATE_SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:aws:cloudformation:logical-id,Values=PrivateSubnet*" --query "Subnets[].SubnetId" --output text --region $REGION)

# Private Subnet ID가 없으면 오류 메시지 출력 후 종료
if [ -z "$PRIVATE_SUBNET_IDS" ]; then
  echo "Private Subnet을 찾을 수 없습니다!"
  exit 1
fi

echo "Private Subnet IDs: $PRIVATE_SUBNET_IDS"

# 2. EFS 생성
# EFS 파일 시스템을 생성하고 파일 시스템 ID를 저장
EFS_ID=$(aws efs create-file-system --creation-token $EFS_NAME --tags Key=Name,Value=$EFS_NAME --query "FileSystemId" --output text --region $REGION)

echo "EFS ID: $EFS_ID"

# 3. EFS가 available 상태가 될 때까지 대기
# EFS가 "available" 상태가 될 때까지 대기
echo "EFS가 available 상태가 될 때까지 대기 중..."
while true; do
  EFS_STATE=$(aws efs describe-file-systems --file-system-id $EFS_ID --query "FileSystems[0].LifeCycleState" --output text --region $REGION)
  if [ "$EFS_STATE" == "available" ]; then
    echo "EFS가 available 상태가 되었습니다."
    break
  fi
  echo "현재 상태: $EFS_STATE. 10초 후 다시 확인합니다."
  sleep 10
done

# 4. 각 Private Subnet에 대해 Mount Target 생성
# 각 Private Subnet에 대해 EFS 마운트 타겟을 생성
for SUBNET_ID in $PRIVATE_SUBNET_IDS; do
  echo "Subnet $SUBNET_ID에 Mount Target 생성 중..."
  aws efs create-mount-target --file-system-id $EFS_ID --subnet-id $SUBNET_ID --security-groups $SECURITY_GROUP_ID --region $REGION
done

echo "EFS 생성 및 Mount Target 설정 완료."

# 5. Security Group 수정 (필요한 경우)
# EFS에 접근할 수 있도록 NFS 트래픽을 허용
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP_ID --protocol tcp --port 2049 --source-group $SECURITY_GROUP_ID --region $REGION

echo "Security Group $SECURITY_GROUP_ID에 NFS 포트(2049) 접근 허용 완료."
```

::::