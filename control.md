## Reference Architecture
 - Control Plane: https://docs.pivotal.io/pivotalcf/2-3/refarch/control.html#topology2
 - PAS for AWS: https://docs.pivotal.io/pivotalcf/2-3/refarch/aws/aws_ref_arch.html

## Create a Jumpbox
 - OS : ubuntu 18.04 amd64 20180912
 - Disk: 50GB 정도 할당
 ###  추가 설치 필요
  - bosh cli: https://github.com/cloudfoundry/bosh-cli/releases
  - bosh dependency: https://bosh.io/docs/cli-v2-install/#additional-dependencies 
  ```
  sudo apt-get install -y build-essential zlibc zlib1g-dev ruby ruby-dev openssl libxslt1-dev libxml2-dev libssl-dev libreadline6-dev libyaml-dev libsqlite3-dev sqlite3
  ```
  - terraform >= 0.11.0 : https://www.terraform.io/downloads.html
  - ruby: ``` sudo apt-get update && sudo apt-get install ruby-full ```
  - uaac : ``` gem install cf-uaac ```
  - install bbl : https://github.com/cloudfoundry/bosh-bootloader/releases
  
## bosh VM 배포하기(bbl up: Bosh Bootloader)
bbl cli를 이용해서 control plane의 bosh VM을 자동으로 생성할 것입니다. 추가로 생성되는 bbl-jumpbox vm은 bosh vm으로 접근하는 proxy VM이므로 삭제하면 안됩니다.



 - https://github.com/cloudfoundry/bosh-bootloader
### bbl-user IAM 생성
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "logs:*",
                "elasticloadbalancing:*",
                "cloudformation:*",
                "iam:*",
                "kms:*",
                "route53:*",
                "ec2:*",
                "s3:*"
            ],
            "Resource": "*"
        }
    ]
}
```

### concourse elb에 targetgroup 추가
```
mkdir -p ~/workspace/bbl-1/terraform

cd ~/workspace/bbl-1/terraform
wget https://raw.githubusercontent.com/pivotalservices/concourse-credhub/master/bbl-terraform/aws/concourse-lb_override.tf

```


### bbl up
```
cd ~/workspace/bbl-1

bbl up --debug \
        --aws-access-key-id <bbl-user access-key-id> \
        --aws-secret-access-key <bbl-user secret-access-key> \
        --aws-region <region> \
        --iaas aws \
        --lb-type concourse \
        --name jy-dojo

```
### bbl up 테스트

```
ubuntu@:~/workspace/bbl-1$ eval "$(bbl print-env)" (bbl up을 실행한 폴더에서 다음 명령을 실행)

ubuntu@:~/workspace/bbl-1$ bbl lbs   (bbl up을 실행한 폴더에서 다음 명령을 실행)

Concourse LB: jy-dojo-env-ur-xxxxx-concourse-lb [jy-dojo-xxx.elb.ap-northeast-1.amazonaws.com]

ubuntu@:~/workspace/bbl-1$ bosh env  (jumpbox 아무 경로에서 실행가능)

Using environment 'https://10.0.0.6:25555' as client 'admin'

Name      bosh-bbl-env-urmia-2018-11-05t01-49z
UUID      ee56ab75-8c03-40ab-bf33-aefb2e655c4d
Version   268.0.1 (00000000)
CPI       aws_cpi
Features  compiled_package_cache: disabled
          config_server: enabled
          dns: disabled
          snapshots: disabled
User      admin

Succeeded
