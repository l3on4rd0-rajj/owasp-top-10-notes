---
tags: [kubernetes, containers, devsecops, seguranca, service-mesh, istio, mtls, pod-security]
aliases:
  - Containers
  - Kubernetes Security
  - Pod Security Standards
  - Service Mesh
  - Istio
---

# Containers, Kubernetes e Service Mesh

Nota de apoio para estudos de segurança em containers, Kubernetes, microsserviços, mTLS, Service Mesh e Istio.

## Associações no Obsidian

- Hub: [[Containers e Orquestração]]
- Docker: [[Docker — Guia Rápido (Instalação, Comandos e Dockerfile)]], [[docker compose]]
- Kubernetes local: [[minikube]]
- Segurança de APIs: [[API Security]], [[security-misconfiguration|API8:2023 — Security Misconfiguration]]
- Riscos operacionais: [[Unrestricted Resource Consumption|API4:2023 — Unrestricted Resource Consumption]], [[Vazamentos e Tokens de API]]

Tags relacionadas: #kubernetes #containers #devsecops #service-mesh #istio #mtls #pod-security

---

## Pod Security Standards (PSS)

Os **Pod Security Standards (PSS)** no Kubernetes definem três níveis de segurança para controlar o que pode ser executado em Pods. Na documentação oficial, esses níveis também aparecem como políticas.

| Nível | Restrição | Uso indicado |
| --- | --- | --- |
| `privileged` | Política mais permissiva. Permite privilégios amplos e escalonamentos conhecidos. | Workloads de sistema ou infraestrutura administrados por operadores confiáveis do cluster. |
| `baseline` | Restrição mínima para bloquear escalonamentos de privilégio conhecidos. | Aplicações não críticas, times de desenvolvimento e workloads que ainda precisam de compatibilidade ampla. |
| `restricted` | Política mais rígida, alinhada com práticas atuais de hardening de Pods. | Aplicações críticas ou ambientes com maior exigência de segurança. |

Em resumo:

- `privileged`: máxima compatibilidade, menor controle.
- `baseline`: equilíbrio entre adoção e proteção.
- `restricted`: maior segurança, com possível custo de compatibilidade.

Esses níveis ajudam a garantir que os Pods criados no cluster sigam um padrão mínimo antes de serem aceitos.

---

## Pod Security Admission (PSA)

O **Pod Security Admission (PSA)** é o mecanismo nativo do Kubernetes que aplica os **Pod Security Standards**.

Durante o processamento de uma requisição na API do Kubernetes, uma das etapas é a passagem pelos **Admission Controllers**. O controller de Pod Security verifica se o Pod está compatível com o padrão configurado para o namespace.

Se um PSS for aplicado a um namespace, qualquer tentativa de criação de Pod nesse namespace será avaliada pelo PSA.

### Modos de aplicação

| Modo | Comportamento |
| --- | --- |
| `enforce` | Rejeita a criação do Pod se ele violar a política. |
| `audit` | Permite a criação do Pod, mas registra a violação nos logs de auditoria. |
| `warn` | Permite a criação do Pod, mas mostra um aviso para o usuário. |

### Sintaxe das labels

```text
pod-security.kubernetes.io/<MODO>=<NIVEL>
```

Exemplo aplicando o nível `baseline` em modo `enforce`:

```text
pod-security.kubernetes.io/enforce=baseline
```

Também é possível fixar a versão da política:

```text
pod-security.kubernetes.io/enforce-version=v1.30
```

### Aplicando em um namespace

Exemplo para um namespace chamado `example-namespace`:

```bash
kubectl label --dry-run=server --overwrite namespace example-namespace \
  pod-security.kubernetes.io/enforce=baseline
```

O `--dry-run=server` é recomendado para validar se os Pods existentes seriam aceitos com a nova política. Se o comando retornar sem alertas, aplique a label de verdade:

```bash
kubectl label --overwrite namespace example-namespace \
  pod-security.kubernetes.io/enforce=baseline
```

### Observações de configuração

Além das labels por namespace, o Admission Controller de Pod Security pode ser configurado no nível do cluster. Isso permite definir comportamentos padrão e exceções, por exemplo:

- namespaces isentos;
- usuários autenticados isentos;
- valores padrão para `enforce`, `audit` e `warn`;
- versão padrão das políticas.

Esse controle é especialmente importante em ambientes com múltiplos times ou workloads com requisitos de segurança diferentes.

---

## Arquitetura de microsserviços

Uma arquitetura de **microsserviços** separa uma aplicação em serviços menores, independentes e normalmente executados como workloads containerizados.

Exemplos:

- Plataforma de streaming:
  - streaming;
  - recomendações;
  - perfis;
  - catálogo.
- Loja online:
  - web app;
  - carrinho;
  - pedidos;
  - consulta de produtos.

Esse modelo contrasta com uma arquitetura monolítica, onde grande parte da lógica da aplicação fica concentrada em uma única base de código.

---

## Problemas comuns em microsserviços

### Comunicação pod-to-pod

Em muitos clusters, o tráfego externo entra por HTTPS, passa por load balancer, ingress controller e firewall, mas a comunicação interna entre Pods continua usando HTTP ou outro protocolo sem criptografia.

Isso significa que o perímetro pode estar protegido, enquanto o tráfego interno do cluster ainda circula sem confidencialidade.

Em ambientes cloud, o risco aumenta porque os serviços podem estar distribuídos entre zonas de disponibilidade diferentes. Nesses casos, tráfego interno sem criptografia pode atravessar partes da infraestrutura compartilhada do provedor.

### Necessidade de mTLS

O **mTLS (Mutual Transport Layer Security)** fornece autenticação mútua entre cliente e servidor.

Diferença prática:

| TLS tradicional | mTLS |
| --- | --- |
| O servidor apresenta certificado. | Cliente e servidor apresentam certificados. |
| O cliente valida a identidade do servidor. | Ambos validam a identidade um do outro. |
| Normalmente usa CA pública ou corporativa. | A organização pode operar sua própria CA para os serviços internos. |

Com mTLS, a comunicação entre serviços passa a ter:

- autenticação mútua;
- criptografia;
- redução de risco de interceptação;
- melhor base para políticas de identidade entre serviços.

---

## Complexidade operacional

Cada microsserviço costuma precisar de lógica que não é diretamente ligada à regra de negócio:

- configuração de comunicação;
- segurança entre serviços;
- retry;
- timeout;
- observabilidade;
- tracing;
- métricas;
- descoberta de serviços.

Quando essa lógica é implementada manualmente em cada aplicação, surgem problemas:

- aumento de esforço para os times de desenvolvimento;
- inconsistência entre serviços;
- maior chance de erro de configuração;
- dificuldade de troubleshooting;
- complexidade maior conforme novos serviços são adicionados.

É nesse ponto que entra o conceito de **Service Mesh**.

---

## Service Mesh

Um **Service Mesh** é uma camada de infraestrutura dedicada a controlar e gerenciar a comunicação serviço-a-serviço.

A ideia central é separar:

- **lógica de negócio**, que continua dentro do microsserviço;
- **lógica operacional e de segurança**, que passa a ser tratada pela malha de serviços.

### Como funciona

O Service Mesh normalmente adiciona um proxy ao lado de cada serviço. Esse proxy é chamado de **sidecar**.

O sidecar executa em paralelo ao container principal e passa a cuidar de funções como:

- roteamento;
- mTLS;
- retries;
- timeouts;
- métricas;
- tracing;
- políticas de tráfego;
- controle de comunicação entre serviços.

Os proxies sidecar interconectados formam a malha de serviços.

### Benefícios para DevSecOps

Para desenvolvimento:

- menos código operacional dentro da aplicação;
- foco maior na regra de negócio;
- menor duplicação entre serviços.

Para segurança:

- comunicação serviço-a-serviço com mTLS;
- políticas consistentes;
- redução de erros manuais;
- melhor visibilidade do tráfego interno.

Para operação:

- métricas padronizadas;
- tracing distribuído;
- controle de retries e timeouts;
- escalabilidade mais previsível.

Ao gerenciar microsserviços em Kubernetes, um Service Mesh deve ser considerado sempre que segurança, observabilidade e controle de tráfego interno forem relevantes.

---

## Istio

O **Istio** é uma implementação popular de Service Mesh para Kubernetes. Outras opções incluem **Linkerd** e **HashiCorp Consul**.

O Istio é útil para arquiteturas que precisam de:

- mTLS entre serviços;
- controle fino de tráfego;
- políticas de autorização;
- observabilidade;
- segurança por padrão;
- integração forte com Kubernetes.

A arquitetura do Istio é dividida em dois planos:

- **Data Plane**;
- **Control Plane**.

---

## Data Plane: Envoy Proxies

No Istio, o **Data Plane** é formado por proxies **Envoy** executando como sidecars.

O Envoy é um proxy open source de alto desempenho usado para controlar tráfego de entrada e saída entre serviços.

Ele permite recursos como:

- criptografia mTLS;
- roteamento inteligente;
- métricas;
- tracing;
- retries;
- timeouts;
- controle de tráfego entre serviços.

A comunicação entre esses proxies forma o Data Plane da malha.

---

## Control Plane: Istiod

O **Istiod** é o componente principal do **Control Plane** do Istio.

Responsabilidades:

- descoberta de serviços;
- distribuição de configuração;
- gerenciamento de certificados;
- conversão de regras de alto nível em configuração para Envoy;
- integração com mecanismos de descoberta do Kubernetes.

Quando um novo serviço é criado no cluster, o Control Plane detecta esse serviço e fornece as configurações necessárias para que o proxy Envoy opere corretamente.

Observação histórica: antes da versão 1.5, o Control Plane do Istio era composto por componentes separados, como Pilot, Citadel, Mixer e Galley. Esses componentes foram consolidados no Istiod para simplificar operação e configuração.

---

## Fluxo geral do Istio

1. Um serviço é implantado no Kubernetes.
2. O Istio injeta ou associa um proxy Envoy ao workload.
3. O Istiod distribui configuração, certificados e regras.
4. O tráfego entre serviços passa pelos proxies Envoy.
5. Políticas de segurança, mTLS, observabilidade e controle de tráfego são aplicadas pela malha.

Resultado:

- o **Control Plane** gerencia configuração e certificados;
- o **Data Plane** executa o controle real do tráfego;
- os serviços mantêm foco na lógica de negócio.

---

## Comandos práticos

### Minikube e namespaces

```bash
minikube start
kubectl get namespaces
kubectl get pods
```

### Habilitar injeção automática do Istio

```bash
kubectl label namespace default istio-injection=enabled
```

### Recriar workloads para receber sidecar

```bash
kubectl delete -f microservice-manifest.yaml
kubectl apply -f microservice-manifest.yaml
```

### Aplicar política de mTLS

Edite o arquivo `auth.yaml` para permitir apenas tráfego mTLS em modo `STRICT`.

Exemplo:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
```

Aplicar:

```bash
kubectl apply -f auth.yaml
kubectl describe peerauthentication default
```

---

## Checklist rápido

- [ ] Definir **PSS** por `namespace` com `labels`
- [ ] Usar `--dry-run=server` antes de aplicar `enforce`
- [ ] Preferir `baseline` para adoção inicial
- [ ] Utilizar `restricted` para workloads críticos
- [ ] Criptografar tráfego interno com **mTLS** quando houver comunicação entre serviços
- [ ] Considerar **Service Mesh** em arquiteturas com muitos microsserviços
- [ ] Validar injeção de `sidecar` ao utilizar **Istio**
- [ ] Revisar policies de autorização além do **mTLS**
- [ ] Correlacionar riscos com [[security-misconfiguration|misconfiguration]]
- [ ] Verificar exposição de segredos
- [ ] Avaliar consumo indevido de recursos