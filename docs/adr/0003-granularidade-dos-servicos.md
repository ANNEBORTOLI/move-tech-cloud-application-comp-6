# ADR 0003: Granularidade da Aplicação — Monolito Modular versus Microsserviços

- **Status:** Aprovado
- **Data:** 2026-08-08
- **Stakeholders:** Tech Lead, Arquiteto de Software, Product Owner, DevOps

---

## Contexto

Durante a fase inicial do projeto MoveTech, precisávamos decidir a granularidade arquitetural da aplicação. O sistema gerencia um domínio simples (CRUD de veículos) com potencial de expansão futura para gestão de clientes, reservas, faturamento e notificações.

As opções consideradas foram:

1. **Monolito Modular:** Aplicação única Spring Boot com separação em camadas (Controller, Service, Repository) e pacotes por domínio. Deploy como único container.
2. **Microsserviços desde o início:** Separar cada domínio futuro em serviços independentes com bancos dedicados.
3. **Arquitetura Híbrida:** Começar como monolito e extrair módulos gradualmente.

A equipe é pequena (2-3 desenvolvedores) e o prazo para o MVP é curto.

---

## Decisão

**Optamos por implementar como Monolito Modular em Camadas (Layered Modular Monolith), com deploy como container único em ECS Fargate.**

### Características da Decisão:

- **Estrutura de Pacotes por Domínio:** O código está organizado da seguinte forma:
  br.com.fiap.movetech/
  ├── controller/
  │ └── VehicleController.java (REST endpoints)
  ├── service/
  │ └── VehicleService.java (lógica de negócio)
  ├── repository/
  │ └── VehicleRepository.java (acesso a dados)
  └── entity/
  └── Vehicle.java (modelo de domínio)

- **Banco de Dados Compartilhado:** PostgreSQL único via RDS
- **Build Único:** Maven gera um JAR, Docker gera uma imagem
- **Deploy Único:** Uma Definição de Tarefa com um container
- **Número Desejado de Tarefas:** 1 (dev)

### Critérios para Reavaliação Futura:

| Gatilho               | Métrica                                                   |
| --------------------- | --------------------------------------------------------- |
| Tamanho do Time       | Mais de 6 desenvolvedores ativos                          |
| Tempo de Build/Deploy | Acima de 15 minutos                                       |
| Carga por Domínio     | Módulo específico recebe 5x mais tráfego que os demais    |
| Ciclos de Release     | Diferentes módulos exigem deploys independentes           |
| Falhas em Cascata     | Bug em um módulo derruba funcionalidades não relacionadas |

---

## Consequências

### ✅ Positivas

- **Velocidade de Desenvolvimento:** Time pequeno entrega features rapidamente sem coordenação entre serviços.
- **Simplicidade Operacional:** Deploy único, logging centralizado, debugging simplificado (sem tracing distribuído).
- **Menor Custo Inicial:** Apenas 1 task ECS (0.25 vCPU), 1 banco RDS, sem necessidade de service mesh ou message broker.
- **Transações ACID:** Operações entre entidades usam transações locais, sem necessidade de Sagas.
- **Onboarding Rápido:** Novos desenvolvedores entendem o sistema completo em horas.

### ❌ Negativas

- **Escalabilidade Limitada:** Todo o monolito escala junto. Se apenas o módulo de veículos recebe carga, o container inteiro é replicado.
- **Acoplamento Potencial:** Sem disciplina, serviços podem depender de implementações internas de outros módulos.
- **Deploy Mais Arriscado:** Uma mudança pequena exige rebuild e deploy completo (mitigado com CI/CD de ~5 min).
- **Dependência Tecnológica Única:** Todos os módulos presos ao Spring Boot 3.2.0 e Java 17.

### ⚠️ Riscos Mitigados

- **Disciplina Arquitetural:** Regras de pacotes e revisão de código para evitar acoplamento.
- **Preparação para Extração:** Interfaces bem definidas entre camadas (Service ↔ Repository).
- **Manifestos Kubernetes Preparados:** Diretório `/k8s/` existe com manifests prontos para futura migração, se necessário.

---

## Alternativas Consideradas

| Alternativa                           | Motivo da Rejeição                                                                                 |
| ------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Microsserviços desde o início**     | Overhead operacional insustentável para MVP com 2 devs. Custo multiplicado por 4x sem necessidade. |
| **Serverless (Lambda + API Gateway)** | Cold start inaceitável para endpoints CRUD síncronos. Custo imprevisível com picos.                |
| **Arquitetura Orientada a Eventos**   | Complexidade desnecessária para fluxos simples de CRUD.                                            |
