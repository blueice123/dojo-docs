## Concourse 클러스터 배포

install guide: https://github.com/cloudfoundry/bosh-bootloader/blob/master/docs/concourse.md#on-aws

## Concourse 설치 설정
```
cd ~/workspace/
ubuntu@~/workspace/ clone https://github.com/concourse/concourse-bosh-deployment
```

### vars.yml 생성
```
external_host: "${EXTERNAL_HOST}"
external_url: "https://${EXTERNAL_HOST}"
local_user:
    username: "<username>"
    password: "<password>"
network_name: 'private'
web_network_name: 'private'
web_vm_type: 'default'
web_network_vm_extension: 'lb'
web_instances: 1
db_vm_type: 'default'
db_persistent_disk_type: '1GB'
worker_vm_type: 'default'
deployment_name: 'concourse'
worker_instances: 1
worker_ephemeral_disk: '100GB_ephemeral_disk'

```
### aws-tls-vars.yml
```
- type: replace
  path: /variables/-
  value:
    name: atc_ca
    type: certificate
    options:
      is_ca: true
      common_name: atcCA

- type: replace
  path: /variables/-
  value:
    name: atc_tls
    type: certificate
    options:
      ca: atc_ca
      alternative_names: [((external_host))]
      organization: atcOrg
```      
### add-credhub-uaa-to-web.yml
```
cd ~/workspace/concourse-bosh-deployment/cluster/operations/
wget https://raw.githubusercontent.com/pivotalservices/concourse-credhub/master/operations/add-credhub-uaa-to-web.yml
```

### add stemcell version 
https://raw.githubusercontent.com/pivotalservices/concourse-credhub/master/versions.yml
```
~/workspace/concourse-bosh-deployment/cluster$ vi ../versions.yml
...

uaa_release_version: 60
uaa_sha: a7c14357ae484e89e547f4f207fb8de36d2b0966

credhub_release_version: 1.9.3
credhub_sha: 648658efdef2ff18a69914d958bcd7ebfa88027a

backup_restore_sdk_release_version: 1.9.0
backup_restore_sdk_sha: 2f8f805d5e58f72028394af8e750b2a51a432466
```

### bosh에 VM 타입 추가
```
cd ~/workspace/concourse-bosh-deployment/cluster/
bosh cloud-config > bosh-cloud-config.yml
vi bosh-cloud-config.yml

vm_types:
- cloud_properties:
    ephemeral_disk:
      size: 102400
      type: gp2
    instance_type: m4.large
  name: disk_100G_type

~/workspace/concourse-bosh-deployment/cluster/$ bosh update-cloud-config ./bosh-cloud-config.yml
```

### prepare-for-concourse.sh (bbl 폴더에서 실행해야됨)
```
export IAAS="$(cat bbl-state.json | jq -r .iaas)"
if [ "${IAAS}" = "aws" ]; then
  export EXTERNAL_HOST="$(bbl outputs | grep concourse_lb_url | cut -d ' ' -f2)"
  export STEMCELL_URL="https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-xenial-go_agent"
elif [ "${IAAS}" = "gcp" ]; then
  export EXTERNAL_HOST="$(bbl outputs | grep concourse_lb_ip | cut -d ' ' -f2)"
  export STEMCELL_URL="https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-xenial-go_agent"
else # Azure
  export EXTERNAL_HOST="$(bbl outputs | grep concourse_lb_ip | cut -d ' ' -f2)"
  export STEMCELL_URL="https://bosh.io/d/stemcells/bosh-azure-hyperv-ubuntu-xenial-go_agent"
fi

bosh upload-stemcell "${STEMCELL_URL}"

```
# stemcell upload
```
$ source prepare-for-concourse.sh

$ bosh stemcells
Using environment 'https://10.0.0.6:25555' as client 'admin'

Name                                     Version  OS             CPI  CID
bosh-aws-xen-hvm-ubuntu-xenial-go_agent  170.2    ubuntu-xenial  -    ami-03266580e7063711a

(*) Currently deployed

1 stemcells
Succeeded
```
# deploy concourse
```
 bosh deploy -d concourse concourse.yml \
      -l ../versions.yml \
      -l vars.yml \
      -o operations/basic-auth.yml \
      -o operations/privileged-http.yml \
      -o operations/privileged-https.yml \
      -o operations/tls.yml \
      -o aws-tls-vars.yml \
      -o operations/web-network-extension.yml \
      -o operations/add-credhub-uaa-to-web.yml \
      -o operations/worker-ephemeral-disk.yml \
      -o operations/scale.yml

/workspace/concourse-bosh-deployment/cluster$ ./deploy-concourse.sh

/workspace/concourse-bosh-deployment/cluster$ bosh vms
Using environment 'https://10.0.0.6:25555' as client 'admin'

Task 9. Done

Deployment 'concourse'

Instance                                     Process State  AZ  IPs        VM CID               VM Type         Active
db/662f97b9-2239-4574-83b1-a2e6b6167840      running        z1  10.0.16.5  i-00563bb71f8611040  default         true
web/7b08e9fe-9876-4714-907b-102d0531eaae     running        z1  10.0.16.4  i-0e4800956a4321b88  default         true
worker/b236035a-f8e6-49b8-b5ab-4e2f4add230a  running        z1  10.0.16.6  i-0d6781715b6408a25  disk_100G_type  true

3 vms

Succeeded
```
# Concourse 파이프라인 테스트
```
fly cli 설치
$ wget https://github.com/concourse/concourse/releases/download/v4.2.1/fly_linux_amd64
Concourse 로그인
$ fly -t dojo login -c https://dojo-mz-concourse-lb-xxxxxxx.elb.ap-northeast-1.amazonaws.com -k -u admin -p '####'
$ fly targets
name  url                                                                             team  expiry            
dojo  https://dojo-mz-concourse-lb-xxxxxxxx.elb.ap-northeast-1.amazonaws.com  main  Fri, 02 Nov 2018 05:35:57 UTC
```
## 샘플 파이프라인
```
~/workspace/concourse-bosh-deployment/cluster$ vi test.yml
jobs:
- name: hello-credhub
  plan:
  - do:
    - task: hello-credhub
      config:
        platform: linux
        image_resource:
          type: docker-image
          source:
            repository: ubuntu
        run:
          path: sh
          args:
          - -exc
          - |
            echo "Hello $WORLD_PARAM"
      params:
        WORLD_PARAM: {{hello}}
```

## 샘플 파이프라인 배포
```
$fly -t dojo set-pipeline -p test -c test.yml -v hello=test
jobs:
  job hello-credhub has been added:
+ name: hello-credhub
+ plan:
+ - do:
+   - task: hello-credhub
+     config:
+       platform: linux
+       image_resource:
+         type: docker-image
+         source:
+           repository: ubuntu
+       run:
+         path: sh
+         args:
+         - -exc
+         - |
+           echo "Hello $WORLD_PARAM"
+     params:
+       WORLD_PARAM: test

apply configuration? [yN]: y
pipeline created!
you can view your pipeline here: https://dojo-mz-concourse-lb-xxxx.elb.ap-northeast-1.amazonaws.co                                                          m/teams/main/pipelines/test

the pipeline is currently paused. to unpause, either:
  - run the unpause-pipeline command
  - click play next to the pipeline in the web ui


Access
https://dojo-mz-concourse-
```
## 샘플 파이프라인 배포 확인
https://dojo-mz-concourse-lb-xxxxxxxx.elb.ap-northeast-1.amazonaws.com 접속 후 
vars.yml에서 설정한 username 및 password로 로그인 후 확인

## Concourse credhub 테스트
```
bbl print-env
ubuntu@:~/workspace/bbl-1$ eval "$(bbl print-env)" 

bbl lbs
ubuntu@:~/workspace/bbl-1$ bbl lbs   (bbl up을 실행한 폴더에서 다음 명령을 실행)
Concourse LB: dojo-mz-env-ur-xxxxx-concourse-lb [dojo-mz-concourse-lb-xxxxxxxx.elb.ap-northeast-1.amazonaws.com]

CONCOURSE_URL 환경 변수 설정
export CONCOURSE_URL="https://dojo-mz-concourse-lb-xxxxxxxx.elb.ap-northeast-1.amazonaws.com"

CONCOURSE CREDHUB으로 접근 하도록 설정
ubuntu@:~/workspace/concourse/concourse-credhub$ source ./target-concourse-credhub.sh
Connecting to Credhub on BOSH Director....
Warning: The targeted TLS certificate has not been verified for this connection.
Warning: The --skip-tls-validation flag is deprecated. Please use --ca-cert instead.
Setting the target url: https://10.0.0.6:8844
Login Successful
Reading environment details from Credhub on BOSH Director....
https://bbl-env-ur-xxxxx-concourse-lb-xxxxx.elb.ap-northeast-1.amazonaws.com:8844
concourse_to_credhub
Plv2Q17463aOAnAknfp7wdC2b33GPf
-----BEGIN CERTIFICATE----- 
-----END CERTIFICATE-----
Connecting to Concourse Credhub...
Warning: The targeted TLS certificate has not been verified for this connection.
Warning: The --skip-tls-validation flag is deprecated. Please use --ca-cert instead.
Setting the target url: https://bbl-env-ur-xxxxx-concourse-lb-xxxxxx.elb.ap-northeast-1.amazonaws.com:8844
Login Successful


ubuntu@:~/workspace/concourse/concourse-credhub$ credhub set -t value -n /concourse/main/hello -v credhub
id: 637fedec-b6f3-4ff4-aa8b-e15cce9f3353
name: /concourse/main/hello
type: value
value: <redacted>
version_created_at: "2018-11-13T07:40:29Z"

ubuntu@:~/workspace/concourse/concourse-credhub$ credhub get -n /concourse/main/hello
id: 637fedec-b6f3-4ff4-aa8b-e15cce9f3353
name: /concourse/main/hello
type: value
value: credhub
version_created_at: "2018-11-13T07:40:29Z"
```
## Credhub 연동 테스트 
```
Credhub 연동 테스트 관련 파이프라인 배포
fly -t concourse sp -p test-credhub -c test.yml

jobs:
- name: hello-credhub
  plan:
  - do:
    - task: hello-credhub
      config:
        platform: linux
        image_resource:
          type: docker-image
          source:
            repository: ubuntu
        run:
          path: sh
          args:
          - -exc
          - |
            echo "Hello $WORLD_PARAM"
      params:
        WORLD_PARAM: ((hello)) <- {{hello}} 대신 ((hello))로 변경해야 Credhub에서 값을 읽음

Credhub 파이프라인 실행
fly -t concourse up -p test-credhub        

Credhub 파이프라인 트리거
fly -t concourse tj -j test-credhub/hello-credhub -w

Successfully pulled ubuntu@sha256:29934af957c53004d7fb6340139880d23fb1952505a15d69a0                                                                         3af0d1418878cb.

running sh -exc echo "Hello $WORLD_PARAM"

+ echo Hello credhub
Hello credhub
succeeded
```



