# Autoescalamento Horizontal de Aplicações Kubernetes

Trabalho Avaliativo 1 da disciplina **PSI512 – Tópicos em Computação em Nuvem (2026)** da Escola Politécnica da Universidade de São Paulo.

## Objetivo

O trabalho avalia experimentalmente o uso do **Horizontal Pod Autoscaler (HPA)** do Kubernetes para o escalamento automático de uma aplicação web em dois ambientes:

- cluster Kubernetes local utilizando **Minikube**;
- cluster Kubernetes gerenciado na AWS utilizando **Amazon EKS**.

A mesma aplicação e os mesmos parâmetros de autoescalamento foram utilizados nos dois ambientes, permitindo comparar o comportamento do HPA durante testes controlados de carga.

## Configuração do experimento

O HPA foi configurado com:

- utilização alvo de CPU: `50%`;
- mínimo de réplicas: `1`;
- máximo de réplicas: `10`;
- CPU request por container: `200m`;
- CPU limit por container: `500m`.

A carga foi gerada por um Pod auxiliar executando BusyBox e realizando continuamente requisições HTTP ao Service da aplicação.

## Estrutura do repositório

```text
.
├── README.md
├── artigo/
│   └── sbc_projeto_intermediario_kubernetes.pdf
│   └── sbc_projeto_intermediario_kubernetes.odt
├── minikube/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── hpa.yaml
├── eks/
│   └── hpa-app.yaml
├── roteiro/
│   ├── roteiro_minikube.md
│   └── roteiro_eks.md
└── evidencias/
    ├── ev01_local_estado_inicial.png
    ├── ev02_local_hpa_scaleup.png
    ├── ev03_local_hpa_estabilizado.png
    ├── ev04_local_hpa_final.png
    ├── ev05_eks_estado_inicial.png
    ├── ev06_eks_scaleup.png
    ├── ev07_eks_estabilizacao.png
    └── ev08_eks_scaledown.png
```

## Implantação local — Minikube

Os manifestos utilizados no ambiente local estão disponíveis na pasta [`minikube`](./minikube).

O procedimento completo de implantação, configuração do Metrics Server, geração de carga e validação do HPA está documentado em:

[`roteiro/roteiro_minikube.md`](./roteiro/roteiro_minikube.md)

## Implantação em nuvem — Amazon EKS

O manifesto utilizado para Deployment, Service e HPA no Amazon EKS está disponível na pasta [`eks`](./eks).

O procedimento de criação e configuração do ambiente AWS, implantação da aplicação, geração de carga, monitoramento, coleta de informações de custo e limpeza dos recursos está documentado em:

[`roteiro/roteiro_eks.md`](./roteiro/roteiro_eks.md)

## Evidências experimentais

As capturas de tela utilizadas para comprovar a execução dos experimentos estão disponíveis na pasta [`evidencias`](./evidencias).

As evidências registram:

- estado inicial da aplicação e do HPA;
- aumento da utilização de CPU;
- scale-up automático;
- funcionamento com múltiplas réplicas;
- retirada da carga;
- scale-down;
- retorno à configuração mínima.

A identificação e descrição individual das evidências EV01–EV08 também estão apresentadas no Anexo A do artigo.

## Artigo técnico

O artigo técnico está disponível nos seguintes formatos:

- [Artigo em PDF](./artigo/sbc_projeto_intermediario_kubernetes.pdf)
- [Arquivo-fonte em ODT](./artigo/sbc_projeto_intermediario_kubernetes.odt)

O arquivo PDF corresponde à versão final do artigo submetida para avaliação.

## Autor

**Joaquin Barreyro**  
Escola Politécnica – Universidade de São Paulo (USP)
