## PAS 업그레이드 순서
https://docs.pivotal.io/pivotalcf/2-3/upgrading/checklist.html#top

1. 버전간 호환성 검사
2. Ops Manager의 서비스 타일 업그레이드
3. Ops Manager 업그레이드
4. PAS 업그레이드

## PAS 업그레이드 파이프라인 구성
1. Ops Manager 업그레이드 파이프라인

### PCF 파이프라인 다운로드
```
https://github.com/pivotal-cf/pcf-pipelines
github clone https://github.com/pivotal-cf/pcf-pipelines.git
```
### params.yml 수정
```
# Existing Ops Manager VM name pattern. This should uniquely filter to a running
# eg.  myenv-OpsMan
existing_opsman_vm_name: jygal-OpsMan az1

# Optional - if your git repo requires an SSH key.
git_private_key: ((git_private_mega_key.private_key))

# Ops Manager Admin Credentials - set during the installation of Ops Manager
# Either opsman_client_id/opsman_client_secret or opsman_admin_username/opsman_a
# If you are using opsman_admin_username/opsman_admin_password, edit opsman_clie
# If you are using opsman_client_id/opsman_client_secret, edit opsman_admin_user
opsman_client_id:
opsman_client_secret:
opsman_admin_username: ((opsman_admin.username))
opsman_admin_password: ((opsman_admin.password))

# If install pipeline has been used then the passphrase is same as the admin pas
opsman_passphrase: ((opsman_admin.password))

# Ops Manager Url - FQDN to access Ops Manager without protocol (will use https)
opsman_domain_or_ip_address: opsman.sampivotal.com

opsman_major_minor_version: ^2\.3\.[0-9]+$ # Ops Manager minor version to track

# Pivotal Net Token to download Ops Manager binaries from https://network.pivota
pivnet_token: ((pivnet_token)) # value must be a Pivotal Network legacy token; U

# AWS params
aws_access_key_id: ((aws_access_key_id))
aws_secret_access_key: ((aws_secret_access_key))
aws_region: ap-northeast-1
aws_vpc_id: vpc-xxxxxxxx

```
### 파이프라인 배포
```
/workspace/pcf-pipelines/upgrade-ops-manager/aws$ fly -t concourse sp -p upgrade-ops-manager -c pipeline.yml -l params.yml
```




2. PAS 업그레이드 파이프라인

