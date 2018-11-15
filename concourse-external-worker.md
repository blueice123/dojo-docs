
### Ops Manager에서 bosh director에 VM 타입 추가
```
bosh cloud-config > bosh-cloud-config.yml
vi bosh-cloud-config.yml

vm_types:
- cloud_properties:
    ephemeral_disk:
      size: 153600
      type: gp2
    instance_type: m4.large
  name: mega-concourse-worker

bosh update-cloud-config ./bosh-cloud-config.yml
```

### Concourse external worker 추가
https://github.com/concourse/concourse-bosh-deployment/tree/master/cluster

#### 파이프라인 external worker 파일 clone
jumpbox에서 concourse-bosh-deployment파이프라인 소스를 opsmanager로 복사합니다.
```
git clone https://github.com/concourse/concourse-bosh-deployment.git
scp -i .ssh/megazonedojo.pem -r ./concourse-bosh-deployment ubuntu@opsman.sampivotal.com:~/concourse-bosh-deployment
```
#### secrets.yml 
bosh credhub에서 worker_key와 tsa_host_key정보를 추출합니다.
```
ubuntu@ip-172-31-28-175:~/workspace$ credhub get -n /bosh-bbl-env-athabasca-2018-11-02t02-47z/concourse/worker_key
id: de46b6a1-94ea-4605-b855-012535b0f730
name: /bosh-bbl-env-athabasca-2018-11-02t02-47z/concourse/worker_key
type: ssh
value:
  private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    xxxxxxx
    -----END RSA PRIVATE KEY-----
  public_key: ssh-rsa xx
  xxxx
  public_key_fingerprint: KyzMcYroTwGS1B1J2W1Ml7t+QDfnEiUNudJZwktLQkA
version_created_at: "2018-11-02T03:46:12Z"

ubuntu@ip-172-31-28-175:~/workspace$ credhub get -n /bosh-bbl-env-athabasca-2018-11-02t02-47z/concourse/tsa_host_key
id: 0f039a6f-36df-47f5-9bf1-8eb29319cbff
name: /bosh-bbl-env-athabasca-2018-11-02t02-47z/concourse/tsa_host_key
type: ssh
value:
  private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    xxx
    -----END RSA PRIVATE KEY-----
  public_key: ssh-rsa xxxx
  public_key_fingerprint: IXlhAOD48KzeTk7vnRJ3seCMzsF1rzbGdL2bfcJy0uI
version_created_at: "2018-11-02T03:46:12Z"
```

추출된 내용을 이용하여 secrets.yml파일을 편집합니다.
```
tsa_host_key:
  public_key: <public_key>
  

worker_key:
  private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
```

### opsmanager vm에서 concourse worker 배포하기
실행전에 확인사항
- bosh cloud-config를 실행하여 결과에 worker_vm_type에 지정된 값이 있는지 확인
- bosh cloud-config를 실행하여 결과에 external_worker_network_name에 지정된 값이 맞는지 확인
```
bosh -e $BOSH_ENVIRONMENT deploy -d concourse-worker external-worker.yml \
-l ../versions.yml \
-v external_worker_network_name='infrastructure' \
-v worker_vm_type=mega-concourse-worker \
-v instances=1 \
-v azs=[ap-northeast-1a] \
-v deployment_name=concourse-worker \
-v tsa_host=bbl-env-ur-xxxxx-xxxxx-lb-17b383bc85ba80ba.elb.ap-northeast-1.amazonaws.com \
-v worker_tags=[external_worker] \
-l secrets.yml
```
### external worker 등록 확인
```
fly -t concourse workers

name                                  containers  platform  tags             team  state    version
17e22bda-1c76-4293-84ed-6f9459e9703a  9           linux     none             none  running  2.1
34db1fb4-dc7a-4b97-9010-d56a352e811e  2           linux     external_worker  none  running  2.1
37cfbf68-1303-43c5-a83f-fe0bf4ace65a  1           linux     none             none  running  2.1
7cc934af-4d35-4d28-9bdf-53561ece8a64  2           linux     none             none  running  2.1
a1320f76-4463-4277-8129-e51cc04c9c84  3           linux     none             none  running  2.1
f29892f4-1755-477f-b9b6-242bd8f3a14e  3           linux     none             none  running  2.1e
```



