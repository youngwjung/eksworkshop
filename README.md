# Learning Series EKS Lab

### Before getting started

#### Let’s provisioning the environment for lab.

1. Create or import a SSH key pair to [EC2 console](https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-2#KeyPair):. 
    1. Choose *Services > EC2 > Network & Security > Key Pairs*, and then choose *Create Key Pair*.
    2. Enter a friendly name like ***workshop-ap-northeast-2***.
    3. Save the private key, make it accessible to your ssh utility of choice, and set the ownership/permissions as needed (for example, `chmod 400 <key name>.pem`).

2. Download `eksworkshop-main.zip` file from git.
        
3. Create an Amazon S3 bucket and upload `./cfn/02_bastion_host.yml`.
    1. Go to  [S3 console](https://s3.console.aws.amazon.com/s3/home?region=ap-northeast-2#).
    2. Create a S3 bucket with preferred name.
    3. Upload `./cfn/02_bastion_host.yml` file as a S3 object.      
    4. Check properties of the object, copy Object URL.
    
4. Modify the **TemplateURL** of Bastion in `./cfn/01_main_vpc_settings.yml` with the uploaded S3 object URL.

```yaml
        Bastion:
        Type: AWS::CloudFormation::Stack
        Properties:
          TemplateURL: https://l2h-workshop.s3.ap-northeast-2.amazonaws.com/02_bastion_host.yml. # <-- 여기를 수정하세요.
          TimeoutInMinutes: 60
          Parameters:
            BastionKeyName: !Ref KeyName
            BastionEC2LatestAmiId: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'
            BastionVPC: !Ref VPC
            BastionPublicSubnet01: !Ref PublicSubnet01
```


5. Create the Lab environment. It will take around 30 minutes.
    1. Open [CloudFormation Console](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/new?templateURL=https://eks-workshop-2021-lab.s3-ap-northeast-1.amazonaws.com/00-eks-vpc-private-subnets.yaml&stackName=WORKSHOP).
    2. Create a CloudFormation Stack by uploading a template file: `./cfn/01_main_vpc_settings.yml`.

        * Stack name must be WORKSHOP.
        * Choose Key pair to enable ssh access to the EC2 instances. 

      It will take around 30 minutes and you will get 5 stacks as below.

        * WORKSHOP
        * WORKSHOP-Bastion-xxxxxxxxxxxxx
        * eksctl-EKS-WORKSHOP-cluster
        * eksctl-EKS-WORKSHOP-addon-iamserviceaccount-kube-system-aws-node
        * WORKSHOP-NG

6. Try to ssh to the bastion host.

    * You can find the public IP of the bastion host on the outputs of CloudFormation Stack `WORKSHOP-Bastion-xxxxxxxxxxxxx`.
    * `ssh -i <YOUR_KEY_PAIR> ec2-user@<PUBLIC_IP>`



**NOTICE**

> You can install all the packages you need to troubleshoot the problem.
> Please explain in as much detail as possible how you solved the problem.




## Lab 1. EKS worker nodes fail to join the cluster.

> A customer reached out to you and asked how to fix the issue currently faced. This customer provisioned the EKS cluster with the CloudFormation stacks you’ve created. _*Please send a reply to customer in English.*_

Hi, AWS Support, 

I created an EKS cluster named EKS-WORKSHOP with *2 worker nodes* located in private subnets and *1 bastion host*. After they were created completely, I used the ssh command to connect to the bastion host WORKSHOP-Bastion-Instance and tried to use the kubectl command to manipulate the cluster. However, I was not able to use kubectl get node to get node information. 

```
$ kubectl get node

No resources found
```

Could you help me to figure out the root cause of the issue and fix it? Thanks. 

**Goal**: The 2 worker nodes can join to the cluster. You can retrieve the nodes by using kubectl get node as below: 

```
$ kubectl get node

NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-128-9.ap-northeast-2.compute.internal    Ready    <none>   10m   v1.21.5-eks-9017834
ip-192-168-192-4.ap-northeast-2.compute.internal    Ready    <none>   10m   v1.21.5-eks-9017834
```


## Lab2. Not able to resolve DNS query inside the cluster.

I have deployed nginx as a deployment and exposed them with the service name called `my-nginx`. And then I created a new pod called busybox in order to access the exposed service with the command $ curl _http://my-nginx_ inside the pod.

```
$ kubectl apply -f ./lab2/deployment-nginx.yml
$ kubectl expose deployment/my-nginx
$ kubectl apply -f ./lab2/busybox.yml
```

However, I couldn’t resolve `my-nginx` dns from the pod ‘busybox’. 

1) Why the dns query doesn’t work and how to fix it?
2) Please explain in detail the workflow of resolving the dns query when executing `curl http://my-nginx` inside the pod.
3) After fixed the dns settings, the dns for `my-nginx` was resolved correctly. However, I could see the dns query failed intermittently and it happened when the dns response has large payload. What is the root cause and how I resolve this issue?


## Lab 3. Create a NLB type service using external controller. 

You need a Network Load Balancer(NLB) for the service lab3-nlb-service. This service will expose the nginx pods which were created by applying the manifest file (deployment_nginx.yml). A NLB should be created by following these rules:

* Should be accessible from the outside of VPC.
* Instance mode
* Health check : 
    * Path :  /
    * Port:  80 (TCP)
    * Interval: 20s
    * The consecutive health check successes required: 2
* Once the traffic comes into a node, it is not going to other node to find the endpoint.

1) Please complete the manifest for the service lab3-nlb-service that meets the above conditions.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lab3-nlb-service
  namespace: default
  annotations:
  
    # < 여기에 필요한 내용을 추가하세요. >
    
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: my-nginx
  
    # < 여기에 필요한 내용을 추가하세요. >    
```

2) Share the outputs of the command :
```
$ kubectl describe serivce lab3-nlb-service
$ kubectl get pod -n kube-system
```

3) By default, kube-proxy uses iptables mode to handle traffics. Please explain how the external traffic finds the endpoint of the service through NLB.  

## Lab 4. aws-node keeps failing to run

I created a worker node and confirmed the worker node and pods are running successfully. 

```
$ kubectl get node

NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-67-206.ap-northeast-2.compute.internal   Ready    <none>   5m57s   v1.21.5-eks-9017834

$ kubectl get po -A

NAMESPACE     NAME                        READY   STATUS    RESTARTS   AGE
default       workload-7c4ff94949-qnx5b   1/1     Running   0          6m50s
kube-system   aws-node-c4w66              1/1     Running   0          5m52s
kube-system   coredns-6dbb778559-6ts92    1/1     Running   0          6m50s
kube-system   coredns-6dbb778559-bznnj    1/1     Running   0          6m50s
kube-system   kube-proxy-2fp28            1/1     Running   0          5m52s
```

However, after I changed some configurations on the worker node and reboot the instance (`ip-192-168-67-206.ap-northeast-2.compute.internal`), it failed to run aws-node-c4w66 pod and other workloads such as `coredns`, `workload` failed to run, too. I have collected the logs from the worker node by eks logs collector and the outputs from some kubectl command.

* Archived log files of worker node: `./lab4/eks_i-0d4c11012c0b21e04_2022-05-08_1557-UTC_0.6.2.tar.gz`
* kubectl command output: `./lab4/lab4_command_output.zip`


Please figure out the root cause of the issue and explain how to fix it.


## Lab 5. Analyzing data from the monitoring tool

The followings are graphs that monitor the inflow of requests for applications when new version of application is deployed in a Kubernetes cluster. The green color shows the requests to v1.0 and the yellow color shows the requests to v2.0. How the application was deployed? Please deduce and explain which deployment strategy was considered in each graph, and how implement each deployment strategy in Kubernetes.

1) 
![image1](https://user-images.githubusercontent.com/9942737/167354689-e7a2b894-315c-4fd1-9573-b68d3f55d9a6.png)

2)
![image2](https://user-images.githubusercontent.com/9942737/167354845-ac69ea7e-03b2-4aed-861c-7b3e0be561e4.png)

3)
![image3](https://user-images.githubusercontent.com/9942737/167354959-8655935f-79c0-4eea-b33e-815996a64852.png)

4) **To reduce down-time of deployment in Kubernetes,** what would you do? Describe how you implement zero-down time deployment in Kubernetes. 


## Submit the assessment report

- 메일 주소: eksworkshop@aws-cse-kr-event.awsapps.com
- 제목: 20220521_[한글명]_[AccountID]
   
아래의 내용을 Text 파일 또는 MS Words 파일에 작성하여 메일에 첨부해주시기 바랍니다.

```
Lab 1. EKS worker nodes fail to join the cluster.

[답변 - Please send a reply to customer in English.]

[Troubleshooting 과정]


Lab 2. Not able to resolve DNS query inside the cluster.

[답변]
1) 
2) 
3) 

[Troubleshooting 과정]


Lab 3. Create a NLB type service using external controller.

[답변]
1) 
2) 
3) 


Lab 4. aws-node keeps failing to run

[이슈원인]
    
[분석과정]

[해결방법]

    
Lab 5. Analyzing data from the monitoring tool

[답변]
1)
2)
3)
4) 

```


## Clean up Lab resources 

If you solved Lab3, please use the following command to delete resources. If not, please ignore this.

```
$ kubectl delete service lab3-nlb-service
```

Then, Please delete CloudFormation stacks in the following orders :

```
1. WORKSHOP-NG
2. eksctl-EKS-WORKSHOP-addon-iamserviceaccount-kube-system-aws-node
3. eksctl-EKS-WORKSHOP-cluster
4. WORKSHOP
```


**NOTICE**

> CloudFormation Stack 삭제과정을 완료하지 않으면 EKS와 관련된 리소스가 남아 과금될 수 있습니다. 
> 관련 리소스가 모두 삭제되었는지 꼭 확인하시기 바랍니다.

