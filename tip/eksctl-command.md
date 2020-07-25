# eksctl command

cluster 생

```text
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/whchoi-cluster.yaml 
```

nodegroup 정보

```text
eksctl get nodegroup --cluster eksworkshop -o json
```

nodegroup stack name 정보 출

```text
eksctl get nodegroup --cluster eksworkshop -o json | jq -r '.[].StackName'
```

nodegroup NodeInstanceARN 정보 출력

```text
eksctl get nodegroup --cluster eksworkshop -o json | jq -r '.[].NodeInstanceRoleARN'
```

