```markdown
# ADR 0001: Utilização de Banco de Dados Gerenciado Fora do Cluster de Containers

- **Status:** Aprovado
- **Data:** 2026-08-07
- **Stakeholders:** Arquiteto Cloud, Tech Lead, DevOps

---

## Contexto

A aplicação MoveTech requer persistência de dados relacional para armazenar informações de veículos. Precisávamos decidir entre duas abordagens:

1. **Banco dentro do cluster:** Executar PostgreSQL como um container adicional no ECS (sidecar pattern) ou em instância EC2 auto-gerenciada.
2. **Banco gerenciado externo:** Utilizar Amazon RDS for PostgreSQL como serviço gerenciado, completamente separado do ciclo de vida dos containers de aplicação.

O sistema está em fase inicial (MVP) com baixa carga, mas planeja-se escalar para produção nos próximos meses.

---

## Decisão

**Optamos por utilizar Amazon RDS for PostgreSQL como banco de dados gerenciado externo ao cluster ECS.**

### Detalhes Técnicos da Decisão:

- Engine: PostgreSQL 16.3
- Instance Class: db.t3.micro (dev) / db.t3.small (prod planejado)
- Storage: 20 GB gp2 com auto-scaling habilitado
- Multi-AZ: Desabilitado em dev, obrigatório para prod
- Backup: Retenção de 7 dias com snapshots automáticos diários
- Credenciais: Gerenciadas via AWS Secrets Manager, injetadas como variáveis de ambiente no container ECS
- Security Group: Acesso exclusivo ao SG do ECS na porta 5432

---

## Consequências

### ✅ Positivas

- **Gerenciamento Simplificado:** Backups automatizados, patching de SO/engine, manutenção gerenciada pela AWS.
- **Alta Disponibilidade Facilitada:** Multi-AZ pode ser habilitado com um clique, sem necessidade de configurar replicação manual.
- **Desacoplamento de Ciclo de Vida:** O banco sobrevive a redeploys, falhas e destruição do cluster ECS. Dados persistem independentemente.
- **Segurança Reforçada:** Rotação automática de credenciais via Secrets Manager, criptografia em repouso (KMS) e em trânsito (TLS).
- **Monitoramento Integrado:** CloudWatch metrics nativas (CPU, conexões, storage, replicação).
- **Conformidade:** Facilita auditorias e compliance com backups gerenciados.

### ❌ Negativas

- **Custo Adicional:** RDS db.t3.micro custa ~$15/mês (R$ 75,00), enquanto um container sidecar seria "gratuito" (apenas consome recursos da task ECS).
- **Latência de Rede:** Comunicação via rede VPC adiciona ~2-5ms comparado a localhost (sidecar). Aceitável para este caso de uso.
- **Vendor Lock-in Moderado:** Uso de features específicas AWS (RDS, Secrets Manager). Migração para outro provedor exigiria adaptações.
- **Dependência de Serviço Externo:** Se o Secrets Manager falhar na inicialização, o container não obtém credenciais (mitigado com retry logic).

### ⚠️ Riscos Mitigados

- **Single Point of Failure em dev:** RDS Single-AZ pode falhar. Para produção, Multi-AZ é obrigatório.
- **Custos de Data Transfer:** Tráfego entre ECS e RDS na mesma AZ é gratuito. Cross-AZ cobra $0.01/GB. Configuramos preferência pela mesma AZ.

---

## Alternativas Consideradas

| Alternativa                        | Motivo da Rejeição                                                                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **PostgreSQL como Sidecar no ECS** | Gerenciamento manual de backups, volume persistente complexo no Fargate (requer EFS), risco de perda de dados em falha da task. |
| **Aurora Serverless v2**           | Custo mínimo mais alto que RDS tradicional para baixa carga, overkill para MVP atual. Pode ser reconsiderado em versão futura.  |
| **DynamoDB**                       | Modelagem de dados relacional inadequada para NoSQL. Relacionamentos e queries complexas seriam inviáveis.                      |
| **EC2 Auto-Gerenciado**            | Sobrecarga operacional desnecessária para equipe pequena. DevOps precisaria gerenciar SO, patches, backups.                     |

---

## Validação

- **Teste de Recuperação:** Realizado restore a partir de snapshot em 8 minutos.
- **Failover Simulado:** Multi-AZ (em staging) completou em 45 segundos.
- **Latência Média:** 8ms entre ECS e RDS na mesma AZ.
```
