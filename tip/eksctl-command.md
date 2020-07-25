# eksctl command

cluster 생성 . 

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

IAM OIDC Provider 연결

```text
eksctl utils associate-iam-oidc-provider --cluster=eksworkshop --approve
```

IAM 역할/정책과 K8s 서비스 계정 연동

```text
eksctl create iamserviceaccount --cluster=eksworkshop --namespace=kube-system --name=alb-ingress-controller --attach-policy-arn=$PolicyARN --override-existing-serviceaccounts --approve
```



