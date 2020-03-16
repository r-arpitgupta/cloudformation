# EKS BASIC CLUSTER WITH DEPLOYMENT SERVER AND DEPENDENT RESOURCES
##Prerequisite
awscli,
aws account with administrator access(programmatic access)
###create instance role and instance profile to access eks cluster
aws cloudformation create-stack --stack-name <STACK_NAME1> --template-body file://../eks_role.template --capabilities CAPABILITY_IAM

check for successful creation
aws cloudformation   describe-stacks --stack-name <STACK_NAME1> --query "Stacks[0].StackStatus"
result of above command should be "CREATE_COMPLETE"

After successful creation of resources run below command and get values of InstanceRole and InstanceProfile
aws cloudformation   describe-stacks --stack-name <STACK_NAME1> --query "Stacks[0].Outputs"

###Run below command to package nested templates
aws cloudformation package --template-file basic-eks-cluster-with-vpc-deployment-server.template  --s3-bucket <S3_BUCKET_NAME>  --output-template-file packaged-template.json --use-json

###configure parameters.json
AWS key pair, deployment server AMI (installed kubectl and other softwares) should be created before configuration
We can configure other parameters as well e.g. environmentName, eks cluster name, worker node configuration
aws cloudformation create-stack --stack-name <STACK_NAME2> --template-body file://packaged-template.json  --parameters file://parameters.json --role-arn <InstanceRoleArn> --capabilities CAPABILITY_IAM  

###login to deployment server using ec2 key pair
aws cloudformation   describe-stacks --stack-name <STACK_NAME2> --query "Stacks[0].Outputs"
get DeploymentInstanceIp value, eksNodeInstanceRole and login using below command
ssh -i ~/.ssh/<EC2 KeyPair name>.pem <username>@<DeploymentInstanceIp>

###update kubeconfig
aws eks update-kubeconfig --name <CLUSTER_NAME> --region <region>

###map instance role for kube-system
touch aws-auth.yaml
####aws-auth.yaml begin
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: <ARN of instance role (not instance profile)>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
####aws-auth.yaml end
modify eksNodeInstanceRole in aws-auth.yaml
kubectl apply -f aws-auth.yaml