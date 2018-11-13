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
name: /concourse/main/install_pcf/db_master_user2
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






