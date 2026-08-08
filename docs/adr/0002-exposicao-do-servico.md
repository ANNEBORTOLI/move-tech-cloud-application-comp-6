```markdown
# ADR 0002: Estratégia de Exposição e Roteamento de Tráfego

- **Status:** Aprovado
- **Data:** 2026-08-07
- **Stakeholders:** Arquiteto Cloud, DevOps, Security Engineer

---

## Contexto

Definir como expor a aplicação MoveTech para acesso externo via internet. As opções consideradas foram:

1. **Application Load Balancer (ALB) + ECS Service:** Load balancer gerenciado AWS na camada 7.
2. **API Gateway + VPC Link:** Exposição via API Gateway com integração privada ao ECS.
3. **CloudFront + ALB:** CDN na frente do load balancer para cache e aceleração global.
4. **Service Connect (ECS Service Mesh):** Roteamento interno entre serviços ECS sem load balancer público.

A aplicação é um backend REST simples.

---

## Decisão

**Optamos por utilizar Application Load Balancer (ALB) Internet-facing como ponto de entrada único para o tráfego externo, roteando para o ECS Service via Target Group.**

### Detalhes Técnicos:

- **Tipo:** ALB (Layer 7 - HTTP/HTTPS)
- **Schema:** Internet-facing (acessível publicamente)
- **Listener:** Porta 80 (HTTP) - HTTPS planejado para prod
- **Target Group:** IP-based target (modo awsvpc do Fargate)
- **Health Check:** Path `/actuator/health`, intervalo 30s, threshold 2
- **Routing:** Simples (único target group), mas preparado para path-based routing futuro
- **Security Group:** Restrito a 0.0.0.0/0 na porta 80 (temporário para dev)
- **Logs:** Access logs habilitados para S3 (debugging)

---

## Consequências

### ✅ Positivas

- **Simplicidade Operacional:** ALB é totalmente gerenciado, sem necessidade de configurar Nginx/Ingress manualmente.
- **Integração Nativa AWS:** Funciona perfeitamente com ECS Fargate (IP targets), sem adaptações ou sidecars.
- **Health Check Robusto:** ALB remove automaticamente targets não saudáveis, evitando tráfego para containers com falha.
- **Escalabilidade Transparente:** Novas tasks ECS são automaticamente registradas no Target Group.
- **Preparado para Evolução:** Suporte futuro a HTTPS (ACM), path-based routing para múltiplos serviços, e WAF para segurança.
- **Baixa Latência:** Roteamento direto, sem overhead de API Gateway ou Lambda Authorizer.
- **Custo Previsível:** ~$18/mês fixo + $0.008/LCU-hora (carga baixa mantém custo estável).

### ❌ Negativas

- **Sem Proteção DDoS Nativa:** ALB sozinho não tem proteção contra ataques volumétricos. Recomenda-se AWS Shield Advanced (ou CloudFront + WAF no futuro).
- **HTTP Apenas (Atual):** Tráfego não criptografado até o ALB. HTTPS com ACM requer configuração adicional (já planejada).
- **Ponto Único de Falha:** Embora o ALB seja altamente disponível (multi-AZ), uma configuração incorreta pode derrubar todo o acesso.
- **Sem Autenticação Integrada:** Diferente do API Gateway, não há suporte nativo a API Keys, Cognito ou Lambda Authorizers.

### ⚠️ Riscos Mitigados

- **Ataques DDoS:** Para produção, recomendamos adicionar CloudFront + WAF na frente do ALB.
- **Certificado HTTPS:** Já provisionado via ACM, aguardando ativação no listener 443.

---

## Alternativas Consideradas

| Alternativa                     | Motivo da Rejeição                                                                                                              |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **API Gateway + VPC Link**      | Custo mais alto ($1.00/milhão de chamadas), latência adicional (~10ms), complexidade desnecessária para CRUD simples.           |
| **CloudFront + ALB**            | Overkill para MVP atual. Será adicionado quando houver necessidade de cache global ou proteção DDoS.                            |
| **Network Load Balancer (NLB)** | Opera na camada 4 (TCP). Perderíamos health check HTTP inteligente e path-based routing futuro.                                 |
| **ECS Service Connect**         | Excelente para comunicação interna entre serviços, mas não expõe tráfego externo sem ALB/NLB adicional.                         |
| **Kubernetes Ingress (Nginx)**  | Os manifestos K8s existem, mas o cluster ainda não está ativo. Manteríamos consistência futura, mas ECS + ALB é mais integrado. |

---

## Validação

- **Teste de Carga:** ALB distribuiu tráfego uniformemente entre 3 tasks ECS, latência adicional < 5ms.
- **Health Check:** Container unhealthy detectado em 45s e removido automaticamente.
- **Failover:** Multi-AZ garante que falha de uma AZ não interrompe o serviço.
```
