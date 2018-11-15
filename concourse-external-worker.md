
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
