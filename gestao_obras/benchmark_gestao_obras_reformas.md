# SaaS de Gestão de Obras --- Especificação técnica v0.4

**Status:** planejamento técnico preliminar · **Data:** 16/10/2026

## 1. Decisões consolidadas

-   Mercado: Brasil, Paraguai e Argentina.
-   Idiomas: português, espanhol e inglês, selecionáveis por usuário.
-   Uma conta SaaS pode conter várias empresas jurídicas; dados de cada
    empresa devem ser isolados.
-   Obras, contratos, equipes, compras, estoques e financeiro pertencem
    a uma empresa jurídica.
-   Contratos: preço fechado, administração, medição e misto.
-   Administração: taxa fixa ou percentual, escolhida por contrato.
-   Contratos podem operar em moedas diferentes; conversões automáticas
    com ajuste manual autorizado.
-   Financeiro gerencial, sem emissão fiscal ou conformidade contábil
    presumida.
-   Estoque por obra; saldo negativo permitido mediante alçada e limite
    configuráveis por empresa.
-   Presença: entrada, saída, atividades e GPS, inclusive com coleta
    offline no Android/iPhone.
-   Portal do cliente no MVP: acesso a informações publicadas e aceite
    autenticado de aditivos, medições e mudanças de escopo, com versão,
    usuário e data.
-   Operação offline no MVP em Android e iPhone.

## 2. Arquitetura proposta

Monólito modular Laravel, API versionada, banco relacional transacional
(PostgreSQL recomendado, a validar), Redis para cache e filas,
armazenamento privado de arquivos, jobs assíncronos, aplicação web
administrativa, portal web e cliente móvel com armazenamento local
persistente.

### 2.1 Hierarquia de isolamento

`SaaSAccount (grupo/assinante) 1:N LegalEntity (empresa jurídica) 1:N Project (obra)`.
`User N:M LegalEntity` via `CompanyMembership`, com papéis e escopos.
Cada registro operacional carrega `company_id`; entidades por obra
também carregam `project_id`. Consultas, políticas, jobs, eventos,
arquivos e relatórios devem validar o contexto empresarial. Um
administrador da conta não recebe automaticamente acesso operacional
irrestrito a todas as empresas sem política explícita.

### 2.2 Módulos

1.  **Platform:** contas, empresas, assinaturas, planos, limites,
    configurações.
2.  **Identity:** usuários, vínculos, papéis, permissões, sessões.
3.  **CRM:** clientes, contatos, fornecedores, colaboradores.
4.  **Commercial:** propostas, orçamentos, versões e itens.
5.  **Contracts:** contratos, componentes, condições, versões, aditivos,
    aprovações.
6.  **Projects:** obras, etapas, cronogramas, tarefas, responsáveis,
    progresso.
7.  **Field:** presença, GPS, diário, atividades, ocorrências, fotos.
8.  **Procurement:** requisições, aprovações, pedidos, recebimentos.
9.  **Inventory:** materiais, saldos por obra, movimentos, reservas
    futuras, alçadas.
10. **Measurements:** medições, itens, evidências, revisão, aceite.
11. **Finance:** contas gerenciais, custos, receitas, pagamentos,
    moedas, taxas.
12. **ClientPortal:** usuários externos, publicações, comentários,
    aceites.
13. **OfflineSync:** dispositivos, lotes, operações idempotentes,
    conflitos.
14. **Shared:** anexos, notificações, auditoria, traduções, relatórios.

## 3. Modelo de dados conceitual

### 3.1 Plataforma e identidade

  -----------------------------------------------------------------------
  Entidade                Campos centrais         Relações
  ----------------------- ----------------------- -----------------------
  `saas_accounts`         id, name, status,       1:N companies
                          plan_id                 

  `companies`             id, saas_account_id,    1:N projects/contracts
                          legal_name, tax_id,     
                          country_code,           
                          base_currency,          
                          timezone, status        

  `users`                 id, name, email,        N:M companies
                          locale, status          

  `company_memberships`   id, company_id,         empresa+usuário
                          user_id, status         

  `roles`                 id, company_id          N:M memberships
                          nullable, name          

  `permissions`           id, code, description   N:M roles

  `membership_roles`      membership_id, role_id  vínculo

  `clients`               id, company_id, name,   1:N contracts
                          legal_id, contact       

  `suppliers`             id, company_id, name,   1:N purchase_orders
                          tax_id                  

  `employees`             id, company_id, user_id 1:N attendance
                          nullable, name, status  
  -----------------------------------------------------------------------

### 3.2 Obras e contratos

  ------------------------------------------------------------------------
  Entidade                Campos centrais          Relações
  ----------------------- ------------------------ -----------------------
  `projects`              id, company_id,          1:N tasks/contracts
                          client_id, code, name,   
                          status, start_at, end_at 

  `project_members`       project_id,              N:M
                          membership_id,           
                          responsibility           

  `project_stages`        id, project_id, name,    1:N tasks
                          sequence, progress       

  `project_tasks`         id, project_id,          1:N activities
                          stage_id, assignee_id,   
                          status, dates            

  `budgets`               id, company_id,          1:N versions
                          project_id, status       

  `budget_versions`       id, budget_id, number,   1:N items
                          currency_code,           
                          approved_at              

  `budget_items`          id, version_id, code,    version
                          quantity, unit,          
                          unit_price, cost         

  `contracts`             id, company_id,          1:N components
                          project_id, client_id,   
                          model, currency_code,    
                          status,                  
                          current_version_id       

  `contract_versions`     id, contract_id,         histórico imutável
                          version_no, snapshot,    
                          created_by, approved_at  

  `contract_components`   id, contract_id, type,   misto/isolado
                          amount, fee_type,        
                          fee_value, pricing_rule  

  `change_orders`         id, contract_id,         1:N approvals
                          version_no, reason,      
                          amount_delta,            
                          days_delta, status       

  `contract_milestones`   id, contract_id, due_at, contrato
                          amount, condition        

  `measurements`          id, contract_id,         1:N items
                          project_id,              
                          period_start,            
                          period_end,              
                          currency_code, status,   
                          version_no               

  `measurement_items`     id, measurement_id,      medição
                          contract_component_id,   
                          quantity,                
                          cumulative_quantity,     
                          amount                   
  ------------------------------------------------------------------------

### 3.3 Campo, estoque e compras

  ---------------------------------------------------------------------------
  Entidade                    Campos centrais         Relações
  --------------------------- ----------------------- -----------------------
  `attendance_records`        id, company_id,         1:N GPS events
                              project_id,             
                              employee_id,            
                              clock_in_at,            
                              clock_out_at, status,   
                              source                  

  `attendance_locations`      id, attendance_id,      registro GPS
                              event_type, lat, lon,   
                              accuracy_m,             
                              captured_at,            
                              received_at             

  `daily_reports`             id, project_id,         1:N activities
                              report_date, author_id, 
                              status                  

  `daily_activities`          id, project_id,         atividade
                              report_id, task_id,     
                              employee_id,            
                              description, hours,     
                              progress                

  `materials`                 id, company_id, sku,    1:N stock
                              name, unit              

  `project_stocks`            id, company_id,         unique(project,
                              project_id,             material)
                              material_id, on_hand,   
                              reserved, version       

  `stock_movements`           id, company_id,         ledger imutável
                              project_id,             
                              material_id,            
                              movement_type,          
                              quantity,               
                              reference_type,         
                              reference_id,           
                              created_by,             
                              authorized_by, reason   

  `stock_approval_policies`   id, company_id,         alçadas
                              role_id,                
                              threshold_quantity,     
                              threshold_value,        
                              enabled                 

  `stock_authorizations`      id, company_id,         auditoria
                              movement_request_id,    
                              approver_id,            
                              before_qty, after_qty,  
                              justification,          
                              approved_at             

  `purchase_requests`         id, company_id,         1:N items
                              project_id,             
                              requester_id, status    

  `purchase_orders`           id, company_id,         1:N receipts
                              project_id,             
                              supplier_id,            
                              currency_code, status   

  `goods_receipts`            id, purchase_order_id,  1:N receipt items
                              project_id,             
                              received_at,            
                              receiver_id, status     
  ---------------------------------------------------------------------------

### 3.4 Moedas, portal, sincronização e auditoria

  -----------------------------------------------------------------------------
  Entidade                      Campos centrais         Relações
  ----------------------------- ----------------------- -----------------------
  `currencies`                  code, decimals, name    referência

  `exchange_rates`              id, base_code,          histórico
                                quote_code, rate,       
                                effective_at, source,   
                                status                  

  `exchange_rate_overrides`     id, company_id,         ajuste auditado
                                exchange_rate_id,       
                                replacement_rate,       
                                reason, approved_by     

  `money_conversions`           id, company_id,         histórico
                                source_amount,          
                                source_currency,        
                                target_amount,          
                                target_currency,        
                                applied_rate,           
                                effective_at, source    

  `financial_entries`           id, company_id,         financeiro
                                project_id,             
                                contract_id,            
                                entry_type, amount,     
                                currency_code, due_at,  
                                paid_at                 

  `client_portal_memberships`   id, user_id, client_id, escopo externo
                                project_id, status      

  `portal_publications`         id, company_id,         conteúdo
                                project_id,             
                                resource_type,          
                                resource_id,            
                                visibility,             
                                published_version       

  `approval_requests`           id, company_id,         1:N decisions
                                project_id,             
                                resource_type,          
                                resource_id,            
                                resource_version,       
                                status, requested_at    

  `approval_decisions`          id, request_id,         aceite
                                actor_id, decision,     
                                comment, decided_at,    
                                evidence_hash,          
                                ip_address              

  `mobile_devices`              id, user_id,            dispositivo
                                device_key, platform,   
                                last_sync_at,           
                                revoked_at              

  `offline_operations`          id, company_id,         idempotência
                                device_id,              
                                operation_uuid,         
                                resource_type, action,  
                                payload_hash, status,   
                                received_at             

  `sync_conflicts`              id, operation_id,       reconciliação
                                reason, server_version, 
                                client_version,         
                                resolution_status       

  `attachments`                 id, company_id,         arquivos
                                project_id, owner_type, 
                                owner_id, storage_key,  
                                checksum, visibility    

  `audit_logs`                  id, company_id,         auditoria
                                actor_id, action,       
                                entity_type, entity_id, 
                                before_hash,            
                                after_hash, occurred_at 
  -----------------------------------------------------------------------------

**Observações de modelagem:** usar UUID/ULID para entidades expostas a
clientes móveis; chaves estrangeiras e índices compostos por
`company_id`; valores monetários em decimal exato, nunca float; saldos
materializados de estoque reconciliáveis com o ledger de movimentos;
localização GPS separada de presença para reduzir exposição de dados
sensíveis.

## 4. Fluxos críticos

### 4.1 Contratos

1.  Orçamento versionado e aprovado.
2.  Contrato criado com modelo, moeda e componentes.
3.  Contrato fechado: valor-base fixo; mudanças por aditivo.
4.  Administração: `fee_type = fixed | percentage`; se percentual,
    calcular sobre base elegível definida no contrato (a base precisa de
    decisão final).
5.  Medição: quantidades ou etapas, evidências e aprovação.
6.  Misto: componentes separados com regras próprias; impedir cobrança
    duplicada.
7.  Aprovação interna, envio ao cliente, aceite autenticado da versão,
    efetivação.
8.  Mudança posterior cria nova versão/aditivo; histórico original não é
    sobrescrito.

### 4.2 Câmbio

1.  Serviço obtém taxas de provedor selecionado, com timestamp, par e
    fonte.
2.  Cache e persistência histórica da taxa; falha do provedor não deve
    inventar taxa.
3.  Ajuste manual exige permissão e justificativa, e pode exigir
    aprovador distinto conforme política.
4.  Contrato e medição preservam moeda original.
5.  Pagamento em outra moeda preserva os dois valores e a taxa
    efetivamente aplicada.
6.  Relatórios convertidos indicam moeda de apresentação, taxa, fonte e
    data; sem taxa disponível, exibir pendência, não somar moedas
    distintas.
7.  Mudança de taxa não reescreve transações históricas já consolidadas.

### 4.3 Presença, GPS e offline

1.  Dispositivo autenticado baixa somente obras e atividades
    autorizadas.
2.  Entrada gera `operation_uuid`, timestamp local, coordenadas,
    precisão GPS e identificação do dispositivo.
3.  Atividades e saída são registradas na base local persistente mesmo
    sem rede.
4.  Fila envia operações ao servidor com chave idempotente e versão do
    registro.
5.  Servidor valida empresa, obra, usuário, permissões, integridade e
    ordem lógica dos eventos.
6.  Resposta por operação: aceita, rejeitada ou em conflito.
7.  Reenvio não duplica ponto ou atividade; conflitos exigem revisão e
    não sobrescrevem silenciosamente dados.
8.  Guardar horário do aparelho e horário de recebimento; GPS não é
    prova infalível de presença.
9.  Coleta GPS deve ser proporcional, informada ao colaborador e
    limitada ao evento de presença, sem rastreamento contínuo por
    padrão.
10. Sincronização em segundo plano no iOS não deve ser presumida; prever
    ação explícita de sincronizar.

### 4.4 Estoque negativo

1.  Saída solicitada para uma obra/material.
2.  Transação bloqueia logicamente o saldo (lock de linha/controle de
    concorrência) e calcula saldo projetado.
3.  Se saldo suficiente, confirma movimento e atualiza projeção.
4.  Se negativo, cria solicitação pendente, identifica alçada e limite
    da empresa.
5.  Aprovador autorizado recebe saldo anterior, saída proposta, saldo
    projetado e justificativa.
6.  Autorização deve ser verificada novamente na confirmação, pois o
    saldo pode ter mudado.
7.  Movimento confirmado é imutável; correções por estorno.
8.  Alertas e relatório de estoque negativo mostram responsáveis e tempo
    até regularização.
9.  Pedidos não geram entrada; somente recebimento confirmado gera
    saldo.
10. No MVP, pedidos de saída podem ser registrados offline como
    intenção, mas não reduzem saldo oficial até validação online.

### 4.5 Portal e aceite

1.  Gestor publica apenas dados explicitamente classificados para
    cliente.
2.  Aditivo, medição ou mudança de escopo gera `approval_request`
    vinculado a uma versão imutável.
3.  Cliente autenticado consulta documento, impactos, anexos e histórico
    autorizado.
4.  Cliente aprova, rejeita ou solicita ajustes; decisão grava
    identidade, data do servidor, versão, hash da evidência e
    comentário.
5.  Revisão do documento após publicação invalida pedido anterior e
    exige novo aceite.
6.  Aprovação do cliente não substitui aprovações internas nem gera
    automaticamente nota fiscal.
7.  A validade jurídica do aceite deve ser analisada por país e tipo de
    documento; assinatura eletrônica poderá ser incorporada
    posteriormente.

## 5. Roadmap de implementação (estimativas relativas, não promessa de prazo)

  ------------------------------------------------------------------------
  Fase              Objetivo          Entregas           Critério de saída
  ----------------- ----------------- ------------------ -----------------
  0 --- Descoberta  Eliminar riscos   PoC offline        testes reais e
  técnica                             iOS/Android,       decisões
                                      câmbio, modelo     registradas
                                      multiempresa       

  1 --- Fundação    Base segura       contas, empresas,  isolamento
                                      usuários, papéis,  multiempresa
                                      idiomas, moedas,   testado
                                      obras              

  2 --- Comercial e Controle          orçamentos         fluxos e
  contratos         contratual        versionados, 4     histórico
                                      modelos, aditivos, íntegros
                                      medições           

  3 --- Operação    Controlar         tarefas, diário,   movimentos
                    execução          compras,           auditáveis
                                      recebimentos,      
                                      estoque, alçadas   

  4 --- Mobile      Operação sem rede presença, GPS,     testes de
  offline                             atividades, fotos, perda/reconexão
                                      fila, conflitos    

  5 --- Financeiro  Fechar ciclo      lançamentos        aprovação
  e portal                            gerenciais,        rastreável
                                      câmbio,            
                                      publicações,       
                                      aceite             

  6 --- Piloto      Preparar          segurança,         piloto real
                    lançamento        observabilidade,   validado
                                      backup, UX, testes 
                                      de carga           
  ------------------------------------------------------------------------

**Dependências:** desenvolver o protocolo de sincronização e as regras
de multimoeda desde a Fase 1, ainda que a interface completa venha
depois. Criar testes automatizados por módulo e de ponta a ponta.

### Backlog P0

-   Isolamento por empresa em APIs, consultas, jobs e anexos.
-   Acesso multiempresa com papéis por vínculo.
-   Contratos fechados, administração, medição e mistos.
-   Versões de contrato e aceite autenticado.
-   Taxas automáticas com override autorizado e histórico.
-   Estoque por obra e saldo negativo com alçada.
-   Presença com entrada, saída, atividade e GPS.
-   Fila offline persistente, UUID idempotente, tratamento de conflitos
    e fotos.
-   Portal do cliente com aditivos, medições e mudanças de escopo.
-   Financeiro gerencial, auditoria e relatórios mínimos.

## 6. Testes e critérios de aceite

-   Usuário de empresa A não acessa registros, anexos ou relatórios da
    empresa B sem vínculo autorizado.
-   Uma conta SaaS pode alternar entre duas empresas sem misturar dados.
-   Contrato misto calcula componentes separadamente e não cobra item
    duas vezes.
-   Ajuste manual de câmbio requer permissão e registra motivo e
    aprovador.
-   Relatório não soma moedas diferentes sem conversão explícita.
-   Reenvio de operação offline não duplica presença, atividade ou foto.
-   Entrada e saída offline sobrevivem ao encerramento do aplicativo.
-   GPS registra precisão e origem; horário do servidor não é
    substituído pelo horário do aparelho.
-   Duas saídas concorrentes não contornam a política de saldo negativo.
-   Aprovador fora da alçada não autoriza saldo negativo.
-   Cliente não vê material interno nem dados de outra obra.
-   Revisar um aditivo aprovado exige nova versão e novo aceite.

## 7. Decisões pendentes

### Bloqueadoras de modelagem

1.  **Grupo e empresas:** usuários globais podem ter permissões
    administrativas em todas as empresas? Quais dados podem ser
    consolidados?
2.  **Banco de dados:** PostgreSQL ou MySQL? Isolamento por `company_id`
    em banco compartilhado ou isolamento físico para planos especiais?
3.  **Câmbio:** provedor(es), frequência, moeda-base por empresa,
    política de fallback, arredondamento e autorização de override.
4.  **Administração:** base de cálculo da taxa percentual (custos
    diretos, custos elegíveis, materiais, mão de obra), mínimo, teto e
    incidência em aditivos.
5.  **Medições:** por quantidade, percentual de etapa, marcos ou
    combinação; regras de retenção e adiantamento.
6.  **Presença/GPS:** tolerância de precisão, localização obrigatória ou
    facultativa, autorização de correção, política de privacidade e
    retenção.
7.  **Estoque:** alçada por quantidade, valor monetário ou saldo
    projetado; conversão de unidades; depósitos centrais; reservas.
8.  **Portal:** quem pode aceitar em nome do cliente, níveis de
    aprovação, validade jurídica esperada e política de revogação.
9.  **Offline:** PWA ou híbrido, prazo de retenção local, limite de
    obras, política para aparelho perdido e anexos volumosos.

### Comerciais e de operação

10. Primeiro segmento de mercado e perfil do cliente-piloto.
11. Planos e cobrança SaaS: por empresa, usuário, obra ativa ou uso.
12. Integração de notificações: e-mail, push, WhatsApp.
13. Relatórios obrigatórios no lançamento.
14. Importação de planilhas e dados legados.
15. Integrações fiscais/contábeis futuras e necessidades legais por
    país.
16. Assinatura eletrônica: quando será obrigatória, se aplicável.
17. Política de backups, retenção e residência de dados.

## 8. Próxima etapa

Resolver as decisões bloqueadoras, produzir ERD detalhado, matriz de
permissões, contratos de API, máquina de estados dos contratos/medições,
protocolo de sincronização offline e histórias de usuário com critérios
de aceite por módulo. Priorizar uma prova de conceito de presença+foto
offline em Android e iPhone e uma prova de contrato
misto+multimoeda+aceite.
