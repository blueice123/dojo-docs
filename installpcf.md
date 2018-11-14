## Concourse 파이프라인을 통한 PAS 배포
https://github.com/pivotal-cf/pcf-pipelines
### PCF 파이프라인 다운로드
```
github clone https://github.com/pivotal-cf/pcf-pipelines.git
```

### PAS를 구성할 리전의 NAT 인스턴스 AMI ID 확인
https://github.com/pivotal-cf/terraforming-aws/commit/64b0971ee0971e6b4b34c6ff984627f046547c0c

### 파이프라인 params.yml
```
cd ~/workspace/pcf-pipelines/install-pcf/aws/
~/workspace/pcf-pipelines/install-pcf/aws/ vi params.yml
```
#### amis_nat: NAT 인스턴스 AMI ID 
- tokyo: ap-northeast-1 = "ami-03cf3903"
- seoul: ap-northeast-2 = "ami-8e0fa6e0"
#### IAM 키 준비
bbl-user와 동일한 권한 + RDS 전체 권한을 부여

### aws_cert_arn
#### 방법1: AWS Certificate manager에서 생성한다.  등록한 도메인:
```
 <pcfdomain>
 *.<pcfdomain>
 *.system.<pcfdomain>
 *.apps.<pcfdomain>
```
등록후 DNS인증방법을 선택
#### 방법2: 직접 위 \<pcfdomain\>에 대한 인증서를 생성하여 AWS Certificate manager에 import한다.

### aws_key_name
AWS EC2 키페어 생성 후 키페어명 입력

### PCF 구성요소 접속 계정 생성
```
 ubuntu@  credhub generate -t user -n /concourse/main/install_pcf/db_master_user  --username=master_db_user -l 15
id: ac2045d5-b68a-4edb-a12a-597ad7057c57
name: /concourse/main/install_pcf/db_master_user
type: user
value: <redacted>
version_created_at: "2018-11-13T08:54:07Z"

ubuntu@ credhub get -n /concourse/main/install_pcf/db_master_user
id: 78c56457-a654-439c-90bc-a3c1b2d1bcf7
name: /concourse/main/install_pcf/db_master_user
type: user
value:
  password: 3fv87mtK1NaZxx
  password_hash: $6$MI3n6Llo$k1RcnhcFXYFCSOHnmNZuwUj9.xwdlSnG2yu.0HzkPQmoqq2jo5TCrnzeniIyG2uG0JZObBGDuvFjdweqDkXeI.
  username: masterdbuser
version_created_at: "2018-11-06T03:28:59Z"
```

### PCF 구성요소 접속 계정 생성 후 params.yml을 아래와 같이 수정
```
# ERT Database Credentials - Required
db_accountdb_password: ((db_master_user.password))
db_accountdb_username: ((db_master_user.username))
db_app_usage_service_password: ((db_master_user.password))
db_app_usage_service_username: ((db_master_user.username))
db_autoscale_password: ((db_master_user.password))
db_autoscale_username: ((db_master_user.username))
db_ccdb_password: ((db_master_user.password))
db_ccdb_username: ((db_master_user.username))
db_diego_password: ((db_master_user.password))
db_diego_username: ((db_master_user.username))
db_locket_password: ((db_master_user.password))
db_locket_username: ((db_master_user.username))
db_networkpolicyserverdb_password: ((db_master_user.password))
db_networkpolicyserverdb_username: ((db_master_user.username))
db_nfsvolumedb_password: ((db_master_user.password))
db_nfsvolumedb_username: ((db_master_user.username))
db_notifications_password: ((db_master_user.password))
db_notifications_username: ((db_master_user.username))
db_routing_password: ((db_master_user.password))
db_routing_username: ((db_master_user.username))
db_silk_password: ((db_master_user.password))
db_silk_username: ((db_master_user.username))
db_uaa_password: ((db_master_user.password))
db_uaa_username: ((db_master_user.username))

# RDS Master Credentials - Required
db_master_password: ((db_master_user.password))
db_master_username: ((db_master_user.username))
```
### PCF 도메인 적용 (사전 도메인 등록 필요)
```
# Domain Names for ERT
pcf_ert_domain: sampivotal.com # This is the domain you will access ERT with, for example: pcf.example.com.
system_domain: system.sampivotal.com # e.g. system.pcf.example.com
apps_domain: apps.sampivotal.com # e.g. apps.pcf.example.com
```

### PCF 버전 
```
# PCF Elastic Runtime minor version to track
ert_major_minor_version: ^2\.3\.[0-9]+$ # ERT minor version to track (e.g ^2\.1\.[0-9]+$ will track 2.0.x versions)
```
### git private key 생성 
```
주의사항: params.yml 내 키명과 동일하게 만들면 파이프라인을 실행할때 에러 발생함
ubuntu@:~/workspace/dojo-concourse/concourse-credhub$ credhub generate -t ssh -n /concourse/main/git_private_mega_key
id: fa01f2a8-68eb-4971-8f7c-12ef2932c523
name: /concourse/main/git_private_mega_key
type: ssh
value: <redacted>
version_created_at: "2018-11-14T02:23:37Z"
```

### git private key 설정
```
# Optional - if your git repo requires an SSH key.
git_private_key: ((git_private_mega_key.private_key))
```

### MySQL 모니터링 알림 설정 
```
필수 값이며 없으면 아무 값이라도 넣어줘야 함
# Email address for sending mysql monitor notifications
mysql_monitor_recipient_email: jygal@megazone.com
```

### Ops Manger 계정 생성
```
credhub generate -t user -n /concourse/main/opsman_admin  --username=opsman_admin -l 15
```
### Ops Manager 계정 설정
```
# Operations Manager credentials
# opsman_admin_username/opsman_admin_password needs to be specified
opsman_admin_username: ((opsman_admin.username))
opsman_admin_password: ((opsman_admin.password))
```

### Ops Manager 도메인 설정
```
# The domain to access Operations Manager;
opsman_domain_or_ip_address: opsman.sampivotal.com #This must be your pcf_ert_domain with "opsman." as a prefix. For example, opsman.pcf.example.com
```

### PEM 키 생성
```
credhub set -t rsa -n /concourse/main/install_pcf/mega_pem -p <PEM 키 파일 경로>
```

### PEM 키 설정
```
# Private Key of the keypair uploaded to AWS to be used for Operations Manager, NAT VMs.
PEM: ((mega_pem.private_key))
```

### Pivnet 토큰 설정
```
아래의 주소로 들어가면 pivnet 토큰 확인 할 수 있음
# Pivnet token for downloading resources from Pivnet. Find this token at https://network.pivotal.io/users/dashboard/edit-profile
pivnet_token: ((pivnet_token))
```

### Route53 Zone ID 설정
```
위의 설정된 IAM이 읽거나 쓸 수 있는 Zone ID여야함
지정한 Zone ID가 도메인을 publish 할 수 있어야 함
# Route53 zone to add records to
ROUTE_53_ZONE_ID: ZKJT4XXXRQ8P
```

### S3 버킷 설정
```
S3 버킷은 미리 생성 및 버저닝을 설정해줘야 하고 테라폼 상태가 저장됨
# For terraform state file (http://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region)
S3_ENDPOINT: https://s3.ap-northeast-1.amazonaws.com
S3_OUTPUT_BUCKET: mXXXXXXo
```

### security_acknowledge 값을 X로 설정
```
# Setting appropriate Application Security Groups is critical for a secure
# deployment. Change the value of the param below to "X" to acknowledge that
# once the Elastic Runtime deployment completes, you will review and set the
# appropriate application security groups.
# See https://docs.pivotal.io/pivotalcf/opsguide/app-sec-groups.html
security_acknowledgement: X
```

### 플랫폼 자원 이름에 prefix 설정
```
# Prefix to use for Terraform-managed infrastructure, e.g. 'pcf-terraform'
# Must be globally unique.
terraform_prefix: jygal
```

### PCF 파이프라인 실행
```
concourse 로그인
~/workspace/pcf-pipelines/install-pcf/aws$ fly -t concourse login --concourse-url https://dojo-mz-concourse-lb-xxxxxxxx.elb.ap-northeast-1.amazonaws.com

PCF 파이프라인 배포
fly -t concourse set-pipeline --pipeline install-pcf -c pipeline.yml -l params.yml

PCF 파이프라인 활성화
fly -t concourse up -p install-pcf   

1. UI에 들어가서 bootstrap-terraform-state job 실행
2. create-infrastructure job 실행
3. 나머지 job은 자동 실행됨
4. 플랫폼 초기화 및 삭제가 필요한 경우 wipe-env job 실행
```






