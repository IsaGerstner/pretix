# 3. Objetivos

O objetivo deste trabalho é projetar e especificar casos de teste funcionais (testes de defeitos) nos níveis unitário, de integração e de sistema (E2E) para a jornada completa da lista de espera no Pretix, aplicando técnicas de Verificação e Validação (particionamento de equivalência, análise de valor limite e cenários de caso de uso/história de usuário). A verificação prévia da suíte existente indicou que o Pretix testa apenas partes isoladas da lista de espera, como envio individual de voucher, cálculo de cota, formulário de inscrição, painel de controle e API REST, sem cobrir a jornada ponta a ponta nem o resgate do voucher da lista no checkout ou no navegador.

## 3.1 História de Usuário

> "Como cliente que chegou tarde e encontrou o ingresso esgotado, quero entrar na lista de espera para ser avisado e conseguir comprar se uma vaga for liberada."

## 3.2 Atores

- **Cliente:** usuário comprador que encontra um ingresso esgotado, entra na lista de espera e, quando uma vaga é liberada e um voucher é emitido, tenta realizar a compra.
- **Organizador do evento:** configura o evento, cotas, prazo de validade do voucher (`waiting_list_hours`), limite de inscrições por usuário (`waiting_list_limit_per_user`) e desativação automática (`waiting_list_auto_disable`).
- **Sistema (Pretix):** gerencia a ordenação da fila, a distribuição automática de vagas, a emissão e expiração de vouchers, o controle de duplicidade/anonimização e o fluxo de checkout.

## 3.3 Pré-condições

- Evento publicado (`live = True`) com vendas abertas e lista de espera ativada.
- Produto/cota sem vagas disponíveis no momento da inscrição do cliente na lista de espera.

## 3.4 Fluxo Principal

1. O cliente acessa a página do evento e vê que o produto está esgotado.
2. O cliente abre o fluxo da lista de espera do produto, informa seu e-mail e envia a inscrição.
3. Uma vaga é liberada na cota do produto (por cancelamento/expiração de pedido ou ajuste de cota).
4. O sistema emite um voucher de uso único (`max_usages = 1`, `block_quota = True`), válido por `waiting_list_hours` horas, para a entrada elegível na fila.
5. O cliente usa o voucher no processo de compra dentro do prazo, concluindo o pedido e marcando o voucher como resgatado.

## 3.5 Fluxos Alternativos

- **Inscrição duplicada ou além do limite:** novas inscrições para o mesmo e-mail (sem diferenciar maiúsculas de minúsculas) e a mesma combinação de produto, variação e data (`subevent`) que ultrapassem o limite configurado são recusadas.
- **Inscrição anonimizada:** entradas cujo campo de e-mail não contém `@` (condição usada pelo Pretix para identificar inscrições anonimizadas) não recebem voucher.
- **Voucher expirado sem compra:** se o prazo vence sem uso do voucher, a vaga pode voltar a ser distribuída para o próximo inscrito elegível quando a tarefa de distribuição automática (`assign_automatically`) executar novamente (o sistema não dispara o repasse de forma instantânea no momento da expiração).
- **Desativação automática por data:** após a data configurada em `waiting_list_auto_disable`, o sistema interrompe o envio automático, impede novas inscrições e os produtos podem voltar a ficar disponíveis para venda; vouchers já enviados anteriormente continuam ativos.
- **Nenhuma vaga disponível:** sem disponibilidade para o produto/variação (ou sem assentos livres em eventos com assentos), nenhum voucher é enviado.

## 3.6 Regras de Negócio

| ID | Regra de Negócio | Localização no Código do Pretix |
| :--- | :--- | :--- |
| RN01 | A fila é ordenada por prioridade decrescente (`-priority`, maior prioridade primeiro) e, em caso de empate, por data de criação (`created`) e `pk` crescentes (mais antigo primeiro) | `pretix/base/models/waitinglist.py` (`ordering = ('-priority', 'created', 'pk')`) e `assign_automatically()` |
| RN02 | Só é enviado voucher se houver disponibilidade para o produto ou variação; em eventos com mapa de assentos, também exige assentos livres | `send_voucher()` (`"This product is currently not available"`) |
| RN03 | Cada inscrição recebe no máximo um voucher | `send_voucher()` (`"A voucher has already been sent to this person"`) |
| RN04 | O voucher vale para 1 uso (`max_usages = 1`), reserva a vaga (`block_quota = True`) e fica associado ao produto, à variação e, quando aplicável, à data da série (`subevent`) | `send_voucher()` (`max_usages = 1`, `block_quota = True`) |
| RN05 | O voucher vence em `waiting_list_hours` horas (padrão 48h, mínimo 1h); continua válido no instante exato de `valid_until` e só expira quando `valid_until < now()` | `send_voucher()`, `pretix/base/settings.py` e `pretix/base/models/vouchers.py` |
| RN06 | Um mesmo e-mail (comparação *case-insensitive*) pode ter no máximo `waiting_list_limit_per_user` inscrições sem voucher para a mesma combinação de produto, variação e data (`subevent`) (padrão 1) | `clean_duplicate()` e `pretix/base/settings.py` |
| RN07 | Após a data de `waiting_list_auto_disable`, o sistema para de enviar vouchers automaticamente, impede novas inscrições e permite que os produtos voltem à venda; vouchers já emitidos permanecem ativos | `pretix/base/services/waitinglist.py` (`assign_automatically()`) e configurações da lista |
| RN08 | Inscrição anonimizada (e-mail sem o caractere `@`) não recebe voucher | `send_voucher()` (`"This entry is anonymized"`) |

---

# 4. Casos de teste

O projeto dos testes segue a pirâmide de níveis da disciplina: testes unitários para regras isoladas, testes de integração para a interação entre modelos, serviços, banco de dados e views/formulários do Django, e testes de sistema (E2E) para o fluxo no navegador sob a ótica do usuário. As técnicas aplicadas são particionamento de equivalência, análise de valor limite e cenários de caso de uso / história de usuário.

Para evitar ambiguidade em relação ao estado atual do repositório, os cenários abaixo estão separados em duas tabelas: os já cobertos pela suíte existente do Pretix e os propostos para implementação pelo grupo (que ainda serão adicionados nos arquivos `test_t3_waitinglist_unit.py`, `test_t3_waitinglist_integration.py` e `tests/e2e_t3/`).

## 4.1 Cenários Já Cobertos

| ID | Nível | Regra | Técnica | Entrada | Resultado Esperado | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| EX01 | Unitário | RN02 | Valor limite | Cota com `0` vagas disponíveis; chamada a `send_voucher()` | Exceção indicando produto indisponível | Coberto |
| EX02 | Unitário | RN03 | Particionamento | Inscrição que já recebeu voucher; segunda chamada a `send_voucher()` | Exceção impedindo envio duplicado | Coberto |
| EX03 | Unitário | RN05 | Particionamento | Validade customizada (24h); chamada a `send_voucher()` | Voucher criado com validade de 24h | Coberto |
| EX04 | Unitário | RN01 | Particionamento | Inscrições com prioridades distintas; distribuição automática (`assign_automatically`) | Voucher enviado primeiro para a inscrição de maior prioridade (não cobre desempate por `created`/`pk`) | Coberto |
| EX05 | Unitário | RN01, RN02 | Particionamento | Múltiplas vagas liberadas na cota; distribuição automática (`assign_automatically`) | Vouchers distribuídos conforme ordem da fila e vagas | Coberto |
| EX06 | Unitário | RN07 | Particionamento | Data atual posterior a `waiting_list_auto_disable`; distribuição automática | Nenhum voucher enviado automaticamente | Coberto |

## 4.2 Casos de Teste Propostos para Implementação

| ID | Nível | Regra | Técnica | Entrada | Resultado Esperado | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| U01 | Unitário | RN02 | Valor limite | Cota com exatamente `1` vaga disponível (sem assentos); chamada a `WaitingListEntry.send_voucher()` | Voucher criado e vinculado à inscrição (`wle.voucher is not None`) | Implementar |
| U02 | Unitário | RN05 | Valor limite | `waiting_list_hours = 1` (limite mínimo), relógio congelado (`@freeze_time`) e `send_voucher()` | Voucher criado com `valid_until == agora + 1 hora` | Implementar |
| U03 | Unitário | RN06 | Valor limite | Limite `1`; já existe inscrição sem voucher para `cliente@example.com`; chamada a `clean_duplicate(...)` com `Cliente@example.com` (variação de maiúsculas/minúsculas) | `ValidationError` levantada impedindo nova inscrição | Implementar |
| U04 | Unitário | RN06 | Valor limite | `waiting_list_limit_per_user = 2`; e-mail já possui `1` inscrição sem voucher; chamada a `clean_duplicate(...)` | Nova inscrição aceita sem erro (fronteira inferior ao limite configurado acima de 1) | Implementar |
| U05 | Unitário | RN06 | Valor limite | `waiting_list_limit_per_user = 2`; e-mail já possui `2` inscrições sem voucher; `clean_duplicate(...)` | `ValidationError` levantada bloqueando a 3ª inscrição (fronteira do limite acima de 1) | Implementar |
| U06 | Unitário | RN08 | Particionamento | Inscrição cujo campo `email` não contém `@` (ex.: `'anonimizado'`, condição específica de anonimização no código); chamada a `send_voucher()` | `WaitingListException` levantada com mensagem de inscrição anonimizada | Implementar |
| U07 | Unitário | RN06 | Particionamento | Limite `1`; e-mail já possui `1` inscrição que já recebeu voucher para o produto A; para o Produto B `clean_duplicate(...)` | Nova inscrição aceita, pois `clean_duplicate` contabiliza apenas inscrições sem voucher | Implementar |
| I01 | Integração | RN01 | Particionamento | 3 inscrições com mesma prioridade criadas em horários distintos (`created`/`pk`), `1` vaga na cota e execução de `assign_automatically` | Voucher atribuído à inscrição mais antiga (critério de desempate) | Implementar |
| I02 | Integração | RN04 | Particionamento | Voucher válido da lista de espera aplicado ao carrinho e envio da confirmação de checkout (`/checkout/confirm/`) dentro do prazo | Pedido criado no banco (`Order.objects.count() == 1`), confirmação exibida (`.thank-you`) e voucher consumido (`redeemed == 1`) | Implementar |
| I03 | Integração | RN04 | Particionamento | Tentativa de aplicar/confirmar compra com o mesmo voucher já consumido (`redeemed == 1`) em um segundo pedido | Segunda tentativa recusada; nenhum novo pedido criado | Implementar |
| I04 | Integração | RN05 | Valor limite | Relógio congelado no instante exato de `valid_until` (`now == valid_until`); tentativa de confirmação de checkout com o voucher | Pedido criado e voucher consumido, pois o Pretix só considera vencido quando `valid_until < now()` | Implementar |
| I05 | Integração | RN05 | Valor limite | Relógio avançado `1` minuto após `valid_until` (`now == valid_until + 1 min`); tentativa de confirmação de checkout com o voucher | Checkout recusado (alerta de erro `.alert-danger` e `Order.objects.count() == 0`) | Implementar |
| I06 | Integração | RN05, RN01 | Particionamento | Voucher do primeiro inscrito expira sem uso e a tarefa `assign_automatically` é executada novamente | Vaga reaproveitada na nova execução e próximo inscrito elegível da fila recebe um voucher novo | Implementar |
| I07 | Integração | RN06 | Particionamento | Dois `POST` em `/<organizador>/<evento>/waitinglist/?item=<id>` com o mesmo e-mail acima do limite | Segunda inscrição recusada com mensagem de erro na resposta | Implementar |
| S01 | Sistema | Fluxo principal | Caso de uso | No navegador (Playwright), usando a fixture `item_sold_out_with_waitinglist`: produto esgotado, cliente clica em *waiting list*, informa e-mail e envia | Mensagem de sucesso visível na tela (`.alert-success`) e registro criado no banco | Implementar |
| S02 | Sistema | RN04 | Caso de uso | No navegador (Playwright): cenário preparado no banco com cota liberada para `1` vaga e voucher emitido via `send_voucher()`; cliente acessa `/<organizador>/<evento>/redeem?voucher=<codigo>` e conclui o checkout | Fluxo finalizado até a tela de pedido confirmado | Implementar |
| S03 | Sistema | RN05 | Caso de uso | No navegador (Playwright): cenário preparado no banco com voucher emitido e `valid_until` ajustado para o passado; cliente acessa `/<organizador>/<evento>/redeem?voucher=<codigo>` | Mensagem de erro exibida na interface impedindo o resgate/compra | Implementar |