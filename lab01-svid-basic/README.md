# 🪪 Lab 01 — Validação básica de SVID com SPIFFE/SPIRE

> **Progresso:** `Lab 01` → [Lab 02](../lab02-mtls-envoy/README.md) → [Lab 03](../lab03-spiffe-id-authorization/README_lab03.md)

Este é o laboratório de partida. O objetivo é validar que o ambiente SPIRE está funcionando corretamente e que um pod consegue receber sua primeira identidade digital.

O pod acessa a **SPIFFE Workload API** via socket e recebe um **X.509-SVID** emitido pelo SPIRE — um certificado que carrega o SPIFFE ID da aplicação.

---

## 1. 🎯 Objetivo

Validar o fluxo básico de emissão de identidade:

```text
Pod Kubernetes
   ↓
SPIFFE CSI Driver monta o socket
   ↓
Workload acessa o SPIRE Agent via Workload API
   ↓
SPIRE Agent entrega um X.509-SVID
   ↓
Workload recebe um SPIFFE ID
```

Resultado esperado:

```text
SPIFFE ID: spiffe://example.org/ns/default/sa/default
```

---

## 2. 🗺️ Diagrama — Fluxo de emissão do SVID

```mermaid
sequenceDiagram
    participant Pod as 📦 Pod
    participant CSI as 🔌 SPIFFE CSI Driver
    participant Agent as 🔐 SPIRE Agent
    participant Server as 🏛️ SPIRE Server

    Pod->>CSI: solicita volume (csi.spiffe.io)
    CSI-->>Pod: monta socket em /run/spire/sockets/spire-agent.sock

    Pod->>Agent: Workload API — FetchX509SVID()
    Agent->>Server: atesta workload (seletor: k8s:ns + k8s:sa)
    Server-->>Agent: X.509-SVID assinado pela CA (Autoridade Certificadora)

    Agent-->>Pod: SVID (certificado + chave privada + bundle)

    Note over Pod: SPIFFE ID recebido:<br/>spiffe://example.org/ns/default/sa/default
    Note over Pod: Válido por 1 hora — renovado automaticamente
```

**Legenda:**

| Ícone | Elemento | O que é |
|-------|----------|---------|
| 📦 | Pod | Unidade de execução do Kubernetes onde a aplicação roda |
| 🔌 | SPIFFE CSI Driver | Responsável por montar o socket SPIFFE dentro do pod |
| 🔐 | SPIRE Agent | Agente local que valida a workload e entrega o certificado |
| 🏛️ | SPIRE Server | Autoridade central que assina os certificados (CA) |

---

## 3. 📚 Conceitos envolvidos

Antes de executar, vale entender três conceitos fundamentais:

### SPIFFE ID

Identidade única da workload, representada como uma URI. Neste lab:

```text
spiffe://example.org/ns/default/sa/default
```

| Campo | Valor | Significado |
|-------|-------|-------------|
| Trust Domain | example.org | O "domínio" de confiança do ambiente |
| Namespace | default | Namespace Kubernetes do pod |
| ServiceAccount | default | ServiceAccount Kubernetes do pod |

### X.509-SVID

Certificado X.509 emitido pelo SPIRE que carrega o SPIFFE ID da workload. É a "identidade digital" do pod — funciona como um crachá com validade.
Válido por padrão por **1 hora**, renovado automaticamente pelo SPIRE Agent sem precisar reiniciar o pod.

### Workload API

Interface local acessada via **socket Unix** — um arquivo especial que permite comunicação direta entre processos no mesmo host, sem passar pela rede:

```text
/run/spire/sockets/spire-agent.sock
```

---

## 4. ✅ Pré-requisitos

- [ ] Minikube rodando
- [ ] SPIRE instalado no namespace `spire` (Server + Agent + CSI Driver)

Validar:

```bash
kubectl get pods -n spire
```

---

## 5. 🔑 Registrar a Workload Entry no SPIRE Server

> [!IMPORTANT]
> **Este passo é obrigatório.** Sem o registro, o pod não receberá nenhum SVID.

Uma Workload Entry informa ao SPIRE Server qual pod está autorizado a receber qual identidade. Obtenha o UID do node:

```bash
kubectl get node minikube -o jsonpath='{.metadata.uid}'
```

Registre a workload no SPIRE Server (substitua `<NODE_UID>` pelo valor obtido):

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://example.org/ns/default/sa/default \
  -parentID spiffe://example.org/spire/agent/k8s_sat/minikube/<NODE_UID> \
  -selector k8s:ns:default \
  -selector k8s:sa:default
```

Confirme que a entry foi criada:

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry show
```

---

## 6. 📁 Arquivo do lab

```text
lab01-svid-basic/
└── spiffe-client.yaml
```

Cria um pod chamado `spiffe-client` no namespace `default` com a ServiceAccount `default`.

O pod usa a imagem `ghcr.io/spiffe/spire-agent:1.14.5` e executa:

```bash
/opt/spire/bin/spire-agent api watch \
  -socketPath /run/spire/sockets/spire-agent.sock
```

---

## 7. ▶️ Como executar

```bash
kubectl apply -f lab01-svid-basic/spiffe-client.yaml
```

Valide o pod:

```bash
kubectl get pod spiffe-client
```

Resultado esperado:

```text
NAME            READY   STATUS    RESTARTS   AGE
spiffe-client   1/1     Running   0          Xs
```

---

## 8. 🔍 Validar o SVID pelos logs

```bash
kubectl logs spiffe-client -c client
```

Exemplo de saída esperada do `api watch`:

```text
time="..." level=info msg="SVID updated" spiffe_id="spiffe://example.org/ns/default/sa/default"
SPIFFE ID:    spiffe://example.org/ns/default/sa/default
SVID Valid After:  ...T00:00:00Z
SVID Valid Until:  ...T01:00:00Z  ← válido por 1 hora
```

> O SPIRE Agent renova o SVID automaticamente antes da expiração. Não é necessário reiniciar o pod.

---

## 9. 🔍 Validar o SVID manualmente com fetch

```bash
kubectl exec -it spiffe-client -c client -- \
  /opt/spire/bin/spire-agent api fetch \
  -socketPath /run/spire/sockets/spire-agent.sock
```

Resultado esperado:

```text
SPIFFE ID:         spiffe://example.org/ns/default/sa/default
SVID Valid After:  ...
SVID Valid Until:  ...
```

---

## 10. 🏁 O que foi validado

- [x] SPIRE Server está funcionando
- [x] SPIRE Agent está funcionando
- [x] CSI Driver montou o socket corretamente no pod
- [x] Workload acessou a Workload API
- [x] Workload recebeu um X.509-SVID válido
- [x] Identidade SPIFFE foi emitida com sucesso

---

## 11. 🛑 Parar o Lab 01

```bash
kubectl delete -f lab01-svid-basic/spiffe-client.yaml
```

---

## 12. 🔧 Troubleshooting

### Socket não encontrado

```text
no such file or directory: /run/spire/sockets/agent.sock
```

O socket correto neste ambiente é:

```text
/run/spire/sockets/spire-agent.sock
```

Confirme no YAML que esse caminho está sendo usado.

### Pod em CrashLoopBackOff

```bash
kubectl logs spiffe-client -c client --previous
kubectl describe pod spiffe-client
```

### SVID não recebido (sem output no watch)

Verifique se a Workload Entry foi registrada corretamente:

```bash
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry show
```

Se estiver vazio, a entry não foi criada. Repita o passo 5.

### Verificar se o socket foi montado

```bash
kubectl exec -it spiffe-client -c client -- \
  ls -la /run/spire/sockets
```

O esperado é encontrar o arquivo `spire-agent.sock`.

### SVID expirado durante testes

O SVID tem validade de 1 hora por padrão. Se o pod ficar parado por muito tempo, o Agent renova automaticamente. Se o Agent estiver com problemas, reinicie:

```bash
kubectl rollout restart daemonset spire-agent -n spire
```

---

## 13. ➡️ Próximo lab

Após validar a emissão do SVID, siga para:

> [**Lab 02 — Comunicação mTLS com Envoy →**](../lab02-mtls-envoy/README.md)

Nele, o SPIFFE ID será usado para estabelecer comunicação **mTLS** segura entre dois pods, com o Envoy atuando como proxy sidecar em cada um.
