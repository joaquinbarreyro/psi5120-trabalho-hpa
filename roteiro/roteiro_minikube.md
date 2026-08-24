# Roteiro de Implantação e Testes — Minikube

## 1. Objetivo

Este roteiro descreve a implantação e validação de uma aplicação web com
Horizontal Pod Autoscaler (HPA) em um cluster Kubernetes local executado
com Minikube.

O experimento utiliza uma aplicação web com HPA baseado na utilização de
CPU. O HPA foi configurado com utilização alvo de 50%, mínimo de 1 réplica
e máximo de 10 réplicas.

Os manifestos utilizados estão disponíveis na pasta `minikube` do
repositório.

---

## 2. Verificação do ambiente Minikube

Verificar a instalação do Minikube:

```bash
minikube version
```

Verificar os clusters existentes:

```bash
minikube profile list
```

O experimento foi executado utilizando o cluster Minikube denominado
`a5-2025`.

Selecionar o perfil:

```bash
minikube profile a5-2025
```

Verificar o estado do cluster:

```bash
minikube status
```

Verificar o node Kubernetes:

```bash
kubectl get nodes
```

---

## 3. Metrics Server

O HPA necessita de métricas de utilização dos Pods para realizar o
escalamento baseado em CPU. No Minikube, o Metrics Server pode ser
habilitado por meio do addon correspondente:

```bash
minikube addons enable metrics-server
```

Verificar se o componente está em execução:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

Após alguns instantes, validar a disponibilização das métricas:

```bash
kubectl top nodes
```

---

## 4. Criação do namespace

Criar um namespace específico para o experimento:

```bash
kubectl create namespace trabalho-hpa
```

Verificar sua criação:

```bash
kubectl get namespaces
```

Todos os recursos da aplicação utilizados no experimento são implantados
nesse namespace.

---

## 5. Implantação da aplicação

Aplicar o Deployment:

```bash
kubectl apply -f minikube/deployment.yaml
```

O Deployment inicia a aplicação com uma réplica e utiliza os seguintes
recursos por container:

- CPU request: `200m`
- CPU limit: `500m`
- Memory request: `64Mi`
- Memory limit: `128Mi`

Verificar a criação do Pod:

```bash
kubectl get pods -n trabalho-hpa
```

Aguardar até que o Pod apresente estado `Running` e `READY 1/1`.

---

## 6. Criação do Service

Aplicar o Service:

```bash
kubectl apply -f minikube/service.yaml
```

Verificar:

```bash
kubectl get svc -n trabalho-hpa
```

Foi utilizado um Service do tipo `ClusterIP`, pois o gerador de carga é
executado dentro do próprio cluster Kubernetes.

---

## 7. Configuração do HPA

Aplicar o manifesto do Horizontal Pod Autoscaler:

```bash
kubectl apply -f minikube/hpa.yaml
```

Verificar:

```bash
kubectl get hpa -n trabalho-hpa
```

O HPA utilizado possui a seguinte configuração:

- métrica: utilização de CPU;
- utilização alvo: 50%;
- número mínimo de réplicas: 1;
- número máximo de réplicas: 10.

Logo após a criação, o campo `TARGETS` pode apresentar temporariamente
`<unknown>/50%` enquanto as primeiras métricas ainda não estiverem
disponíveis.

Validar as métricas dos Pods:

```bash
kubectl top pods -n trabalho-hpa
```

Após a disponibilização das métricas, verificar novamente:

```bash
kubectl get hpa -n trabalho-hpa
```

Antes da aplicação da carga, deve existir uma única réplica e a utilização
de CPU deve permanecer baixa.

---

## 8. Geração de carga

Criar um Pod auxiliar denominado `load-generator`, utilizando BusyBox:

```bash
kubectl run -n trabalho-hpa load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c \
  'while true; do wget -q -O- http://web-hpa; done'
```

O Pod executa continuamente requisições HTTP contra o Service `web-hpa`,
elevando a utilização de CPU da aplicação.

---

## 9. Monitoramento do scale-up

Em outro terminal, acompanhar continuamente o HPA:

```bash
kubectl get hpa -n trabalho-hpa -w
```

Também podem ser monitorados os Pods:

```bash
kubectl get pods -n trabalho-hpa -w
```

E as métricas de utilização:

```bash
kubectl top pods -n trabalho-hpa
```

Durante o experimento, o aumento da utilização de CPU provocou o
escalamento horizontal da aplicação. Foram observadas as configurações de
1, 4 e 8 réplicas.

Com oito réplicas, a utilização média de CPU permaneceu aproximadamente
entre 52% e 54%, próxima ao alvo de 50% configurado no HPA.

---

## 10. Interrupção da carga

Remover o Pod responsável pela geração de carga:

```bash
kubectl delete pod load-generator -n trabalho-hpa
```

Confirmar sua remoção:

```bash
kubectl get pods -n trabalho-hpa
```

Após a interrupção da carga, acompanhar novamente o HPA:

```bash
kubectl get hpa -n trabalho-hpa -w
```

A utilização de CPU deve cair rapidamente. Entretanto, a redução do número
de réplicas não ocorre imediatamente devido ao comportamento de
estabilização do HPA.

No experimento realizado, foi observada a redução:

```text
8 → 3 → 1
```

O retorno à configuração mínima ocorreu aproximadamente cinco a seis
minutos após a retirada da carga.

---

## 11. Comandos auxiliares de diagnóstico

Para obter informações detalhadas sobre o HPA:

```bash
kubectl describe hpa web-hpa -n trabalho-hpa
```

Para verificar o Deployment:

```bash
kubectl get deployment -n trabalho-hpa
```

Para consultar o consumo atual dos Pods:

```bash
kubectl top pods -n trabalho-hpa
```

Para visualizar todos os recursos principais do namespace:

```bash
kubectl get all -n trabalho-hpa
```

---

## 12. Limpeza do ambiente

Após a conclusão do experimento, os recursos podem ser removidos pela
exclusão do namespace:

```bash
kubectl delete namespace trabalho-hpa
```

Essa operação remove os objetos do experimento associados ao namespace,
incluindo Deployment, Service, HPA e Pods.

O addon Metrics Server pode permanecer habilitado no cluster Minikube para
utilização em outros experimentos.
