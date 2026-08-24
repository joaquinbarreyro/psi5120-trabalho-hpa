# Roteiro de Implantação e Testes — Amazon EKS

## 1. Objetivo

Este roteiro descreve a implantação e validação de uma aplicação web com
Horizontal Pod Autoscaler (HPA) em um cluster Kubernetes gerenciado na
AWS utilizando Amazon Elastic Kubernetes Service (EKS).

O experimento utiliza a mesma aplicação e os mesmos parâmetros empregados
no ambiente Minikube, permitindo comparar o comportamento do HPA nos dois
ambientes.

A região AWS utilizada foi `us-east-1`.

---

## 2. Verificação da conta AWS

O ambiente foi administrado por meio do AWS CloudShell.

Verificar a identidade utilizada:

```bash
aws sts get-caller-identity
```

Verificar inicialmente os clusters EKS existentes:

```bash
aws eks list-clusters --region us-east-1
```

---

## 3. Infraestrutura de rede

Foi criada uma infraestrutura de rede por meio de uma stack do AWS
CloudFormation denominada:

```text
psi5120-hpa-vpc
```

Após a criação da stack, os identificadores da VPC, sub-redes e Security
Group podem ser consultados com:

```bash
aws cloudformation describe-stacks \
  --region us-east-1 \
  --stack-name psi5120-hpa-vpc \
  --query 'Stacks[0].Outputs' \
  --output table
```

No experimento realizado foram utilizadas três sub-redes associadas à VPC
do cluster.

---

## 4. IAM Role do cluster EKS

Para o plano de controle do EKS foi utilizada a IAM Role:

```text
psi5120-hpa-eks-cluster-role
```

A role recebeu a política gerenciada:

```text
AmazonEKSClusterPolicy
```

A associação pode ser verificada com:

```bash
aws iam list-attached-role-policies \
  --role-name psi5120-hpa-eks-cluster-role \
  --output table
```

---

## 5. Criação do cluster EKS

Foi criado o cluster:

```text
psi5120-hpa-eks
```

utilizando Kubernetes versão `1.36`.

Após a criação, verificar seu estado:

```bash
aws eks describe-cluster \
  --region us-east-1 \
  --name psi5120-hpa-eks \
  --query 'cluster.{Version:version,Status:status}' \
  --output table
```

Antes de prosseguir, o estado do cluster deve ser:

```text
ACTIVE
```

---

## 6. IAM Role do worker node

Para o grupo de nodes foi utilizada a IAM Role:

```text
psi5120-hpa-eks-node-role
```

Foram associadas as seguintes políticas gerenciadas:

```text
AmazonEKSWorkerNodePolicy
AmazonEKS_CNI_Policy
AmazonEC2ContainerRegistryPullOnly
```

Verificar as políticas associadas:

```bash
aws iam list-attached-role-policies \
  --role-name psi5120-hpa-eks-node-role \
  --output table
```

---

## 7. Criação do node group

Foi criado o node group:

```text
psi5120-hpa-ng
```

com a seguinte configuração:

- tipo de instância: `t3.small`;
- capacidade: `ON_DEMAND`;
- número mínimo de nodes: 1;
- número desejado de nodes: 1;
- número máximo de nodes: 1;
- volume EBS: 20 GiB.

Após a criação, a configuração pode ser consultada com:

```bash
aws eks describe-nodegroup \
  --region us-east-1 \
  --cluster-name psi5120-hpa-eks \
  --nodegroup-name psi5120-hpa-ng \
  --query 'nodegroup.{InstanceTypes:instanceTypes,DiskSize:diskSize,Scaling:scalingConfig,CapacityType:capacityType}' \
  --output table
```

Antes de prosseguir, o node group deve apresentar estado `ACTIVE`.

---

## 8. Configuração do kubectl

Configurar o acesso do `kubectl` ao cluster:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name psi5120-hpa-eks
```

Verificar a comunicação com o cluster:

```bash
kubectl get nodes
```

O node deve apresentar estado:

```text
Ready
```

Também podem ser verificados os componentes básicos do cluster:

```bash
kubectl get pods -n kube-system
```

---

## 9. Instalação do Metrics Server

O Metrics Server foi instalado utilizando o manifesto oficial do projeto:

```bash
kubectl apply -f \
https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verificar o Pod:

```bash
kubectl get pods -n kube-system | grep metrics
```

Aguardar até que apresente:

```text
1/1 Running
```

Validar a disponibilização das métricas:

```bash
kubectl top nodes
```

No experimento, o node `t3.small` apresentou inicialmente baixa utilização
de CPU antes da implantação da carga de teste.

---

## 10. Criação do namespace

Criar o namespace utilizado no experimento:

```bash
kubectl create namespace trabalho-hpa
```

Verificar:

```bash
kubectl get namespaces
```

---

## 11. Implantação da aplicação e do HPA

O arquivo `eks/hpa-app.yaml` contém os objetos Deployment, Service e
HorizontalPodAutoscaler utilizados no experimento.

Aplicar o manifesto:

```bash
kubectl apply -f eks/hpa-app.yaml
```

O Deployment utiliza por container:

- CPU request: `200m`;
- CPU limit: `500m`;
- Memory request: `64Mi`;
- Memory limit: `128Mi`.

O HPA utiliza:

- utilização alvo de CPU: 50%;
- mínimo de réplicas: 1;
- máximo de réplicas: 10.

Verificar os recursos criados:

```bash
kubectl get pods -n trabalho-hpa
```

```bash
kubectl get svc -n trabalho-hpa
```

```bash
kubectl get hpa -n trabalho-hpa
```

Logo após a criação, o campo `TARGETS` do HPA pode apresentar
temporariamente:

```text
cpu: <unknown>/50%
```

enquanto as primeiras métricas ainda não estiverem disponíveis.

Após alguns instantes, validar:

```bash
kubectl top pods -n trabalho-hpa
```

```bash
kubectl get hpa -n trabalho-hpa
```

Antes da geração de carga, o experimento apresentou uma única réplica e
utilização de CPU próxima de 0%.

---

## 12. Geração de carga

Criar um Pod auxiliar denominado `load-generator`:

```bash
kubectl run load-generator \
  -n trabalho-hpa \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c 'while true; do wget -q -O- http://web-hpa; done'
```

O Pod executa continuamente requisições HTTP contra o Service `web-hpa`,
aumentando a utilização de CPU dos Pods da aplicação.

---

## 13. Monitoramento do scale-up

Acompanhar continuamente o HPA:

```bash
kubectl get hpa -n trabalho-hpa -w
```

Em outro terminal, podem ser acompanhados os Pods:

```bash
kubectl get pods -n trabalho-hpa -w
```

Consultar também as métricas:

```bash
kubectl top pods -n trabalho-hpa
```

Durante o experimento no EKS foram observadas as seguintes quantidades de
réplicas:

```text
1 → 4 → 5 → 8
```

Com oito réplicas, a utilização média de CPU permaneceu
predominantemente entre aproximadamente 58% e 75%.

Para obter informações adicionais sobre as decisões do HPA:

```bash
kubectl describe hpa web-hpa -n trabalho-hpa
```

Durante o experimento foram registrados eventos `SuccessfulRescale`,
confirmando as alterações realizadas automaticamente pelo HPA.

---

## 14. Interrupção da carga e scale-down

Remover o gerador de carga:

```bash
kubectl delete pod load-generator -n trabalho-hpa
```

Continuar acompanhando o HPA:

```bash
kubectl get hpa -n trabalho-hpa -w
```

Após a interrupção das requisições, a utilização de CPU caiu para
aproximadamente 0%. A redução das réplicas ocorreu posteriormente,
seguindo a sequência:

```text
8 → 3 → 1
```

No experimento realizado, o retorno à configuração mínima ocorreu
aproximadamente cinco minutos após a retirada da carga.

Confirmar o estado final:

```bash
kubectl get hpa -n trabalho-hpa
```

```bash
kubectl get pods -n trabalho-hpa
```

```bash
kubectl top pods -n trabalho-hpa
```

---

## 15. Coleta de informações da infraestrutura

A configuração do node group utilizada para a análise de custos foi
consultada com:

```bash
aws eks describe-nodegroup \
  --region us-east-1 \
  --cluster-name psi5120-hpa-eks \
  --nodegroup-name psi5120-hpa-ng \
  --query 'nodegroup.{InstanceTypes:instanceTypes,DiskSize:diskSize,Scaling:scalingConfig,CapacityType:capacityType}' \
  --output table
```

Os volumes EBS em utilização foram consultados com:

```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters Name=status,Values=in-use \
  --query 'Volumes[].{VolumeId:VolumeId,Type:VolumeType,SizeGiB:Size,State:State}' \
  --output table
```

As instâncias EC2 em execução foram consultadas com:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,Type:InstanceType,PublicIPv4:PublicIpAddress,PrivateIPv4:PrivateIpAddress}' \
  --output table
```

No experimento foi utilizada uma instância `t3.small`, um volume EBS gp3
de 20 GiB e um endereço IPv4 público. Não foram utilizados Load Balancer
ou NAT Gateway.

---

## 16. Consulta ao AWS Cost Explorer

Após o experimento, o custo registrado para o período foi consultado por
meio do AWS Cost Explorer:

```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-08-23,End=2026-08-24 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```

No momento da consulta, os custos dos recursos utilizados ainda não
haviam sido consolidados pelo Cost Explorer. Por esse motivo, a análise
apresentada no artigo utiliza uma estimativa baseada nas tarifas públicas
dos serviços AWS efetivamente provisionados.

---

## 17. Limpeza dos recursos

Após a conclusão do experimento, os recursos foram removidos para evitar
a continuidade das cobranças.

### 17.1. Recursos Kubernetes

Remover o namespace do experimento:

```bash
kubectl delete namespace trabalho-hpa
```

### 17.2. Node group

Excluir o node group:

```bash
aws eks delete-nodegroup \
  --region us-east-1 \
  --cluster-name psi5120-hpa-eks \
  --nodegroup-name psi5120-hpa-ng
```

Aguardar sua remoção antes de excluir o cluster.

### 17.3. Cluster EKS

```bash
aws eks delete-cluster \
  --region us-east-1 \
  --name psi5120-hpa-eks
```

### 17.4. Stack de rede

Após a remoção do cluster, excluir a stack:

```bash
aws cloudformation delete-stack \
  --region us-east-1 \
  --stack-name psi5120-hpa-vpc
```

Aguardar a conclusão:

```bash
aws cloudformation wait stack-delete-complete \
  --region us-east-1 \
  --stack-name psi5120-hpa-vpc
```

### 17.5. IAM Roles

Após a remoção dos recursos dependentes, desanexar as políticas das IAM
Roles e excluir as roles `psi5120-hpa-eks-cluster-role` e
`psi5120-hpa-eks-node-role`.

---

## 18. Verificação final da limpeza

Confirmar que não existem clusters EKS:

```bash
aws eks list-clusters --region us-east-1
```

Verificar instâncias EC2 remanescentes:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query 'Reservations[].Instances[].{Id:InstanceId,State:State.Name,Type:InstanceType}' \
  --output table
```

Verificar volumes EBS:

```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters Name=status,Values=available,in-use \
  --query 'Volumes[].{Id:VolumeId,State:State,Size:Size,Type:VolumeType}' \
  --output table
```

Verificar stacks CloudFormation ativas:

```bash
aws cloudformation list-stacks \
  --region us-east-1 \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query 'StackSummaries[].StackName' \
  --output table
```

Ao final do experimento, essas verificações não apresentaram clusters EKS,
instâncias EC2, volumes EBS ou stacks CloudFormation remanescentes
associados ao ambiente criado.
