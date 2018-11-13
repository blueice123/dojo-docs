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







