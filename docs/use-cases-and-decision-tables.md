# Matriz de Casos de Uso, Tabelas de Decisão e Exemplos Executáveis

[Voltar ao Índice Geral](index.md)

---

## 1. Matriz Canônica de Casos de Uso

A matriz formaliza os fluxos operacionais dos atores do ecossistema FitFight, provendo pré-condições, fluxos e garantias de estado pós-condição em nível executável:

| Código | Caso de Uso | Ator Principal | Pré-condições | Fluxo Principal (Passos) | Pós-condições / Garantias |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **UC01** | Cadastro e Cálculo de Fitness Inicial | Novo Atleta | Registro via Google/Strava ou e-mail/senha. | 1. Atleta conclui autenticação.<br>2. Se conectou integração, lê dados históricos disponíveis.<br>3. Aplica $P_{\text{inicial}} = \min(200.0, (N_{30d} \times 5) + (H_{30d} \times 10))$; sem integração: $0.0$ [ESTABELECIDO].<br>4. Inicializa $P_{\text{global}} \leftarrow P_{\text{inicial}}$. | Usuário ativo em `users`; score global inicializado; sem pontos de boas-vindas. |
| **UC02** | Sincronizar Atividade Strava com PRs | Atleta Amador | Conta vinculada ao Strava OAuth2; treino registrado. | 1. Strava envia evento `create` com metadados.<br>2. Backend valida `subscription_id`, `owner_id` e callback secreto [ESTABELECIDO]; POST não possui assinatura criptográfica; enfileira no PGMQ.<br>3. Worker executa `GET /activities/{id}` e usa a resposta como fonte da verdade.<br>4. Para `private=true`, exige `activity:read_all`; sem escopo, marca não verificável e pede reautorização, mantendo pontuação verificada [ESTABELECIDO].<br>5. Avalia tempo em movimento, FC, `pr_count`, `best_efforts[].pr_rank`, `segment_efforts[].pr_rank` e `segment_efforts[].kom_rank`; PR+KOM somam [ESTABELECIDO].<br>6. Calcula pontos e persiste idempotentemente. | Atividade salva por `external_id`; pontuações e desafios elegíveis atualizados. Meta HTTP 200 em menos de 2s [ESTABELECIDO]. |
| **UC03** | Tratar Exclusão/Alteração de Treino Strava | Worker do Sistema | Webhook Strava `aspect_type IN ('update', 'delete')`. | 1. Para todo `update`, sempre busca novamente a atividade e recalcula idempotentemente; `updates` não revela todas as alterações de score.<br>2. Para `delete`, remove a parcela e registra estorno.<br>3. Em desafios `ACTIVE` ou `AUDIT_GRACE`, recalcula soma válida; o placar pode diminuir por correção/exclusão.<br>4. Em desafio concluído, congela o ranking histórico, mas corrige o global. | Correção auditável; replay do mesmo evento não duplica crédito ou estorno. |
| **UC04** | Registrar Bioimpedância e Recomposição | Atleta em Reeducação | Autenticado; cooldown de 48h [ESTABELECIDO]. | 1. Envia peso e dados de composição corporal.<br>2. Se primeira aferição, grava baseline e credita $0.0$ ponto.<br>3. Em aferições seguintes, deriva deltas de massa magra/gorda e zera cada termo separadamente quando $|\Delta|<0.2\text{ kg}$ [ESTABELECIDO].<br>4. Aplica $150\widehat{\Delta M}_{magra}-100\widehat{\Delta M}_{gorda}$ [ESTABELECIDO].<br>5. Atualiza $P_{\text{global}}$ e apresenta informação de saúde, sem diagnóstico médico. | Registro salvo; sinais de perda de massa magra permanecem mesmo se o saldo total for positivo. |
| **UC05** | Detectar Perda Nociva de Massa Magra | Atleta em Dieta Extrema | Nova aferição com $\Delta M_{\text{magra}}\le -0.2\text{ kg}$ (limite inclusivo da banda [ESTABELECIDO]). | 1. Usuário envia medição com redução muscular.<br>2. O termo de massa magra subtrai pontos; calcula-se o saldo líquido com o termo de gordura.<br>3. O sistema sinaliza perda de massa magra mesmo se a redução de gordura atenuar o saldo.<br>4. Recomenda avaliação profissional, sem apresentar diagnóstico clínico. | Penalidade biométrica e alerta informativo. |
| **UC06** | Auditoria de Inatividade Física | Worker do Sistema (Cron) | Regra canônica de 24h consecutivas sem atividade qualificada. | 1. Calcula término da última atividade como `start_date + elapsed_time` (pausas incluídas).<br>2. A cada bloco completo de 24h sem atividade qualificada, aplica a penalidade fixa de −30 pontos [ESTABELECIDO].<br>3. O limiar de 15 min em movimento para qualificar é [ESTABELECIDO].<br>4. Registra prazo e transação idempotente em `points_history`. | Pontuação global auditável; dia civil e Rest Day são alternativa adicional [ESTABELECIDO]. |
| **UC07** | Reconciliação após Sincronização Tardia | Worker do Sistema | `create`, `update` ou `delete` recebido após o instante real da atividade. | 1. Ordena a atividade por `start_date`, calcula término com `elapsed_time` e recalcula a linha temporal a partir do primeiro instante afetado.<br>2. Reverte somente lançamentos de inatividade cujos prazos foram interrompidos pela atividade qualificada; mantém/aplica penalidades de outros blocos completos ainda sem atividade.<br>3. Gera lançamentos compensatórios idempotentes e atualiza o score da atividade. | Nenhum limite civil de 24h; histórico de pontos e penalidades reconciliado e auditável. |
| **UC08** | Criar Desafio e Convites por Link / E-mail | Criador do Desafio | Atleta autenticado. | 1. Configura título (3–255 caracteres), período (mínimo 1 mês), modalidade e participantes (padrão 50, máx 1.000) [ESTABELECIDO].<br>2. Sistema cria desafio e inicializa pontos em $0.0$.<br>3. Envia convite por e-mail/link; HMAC-SHA256 com TTL de 7 dias e uso único [ESTABELECIDO]. | Desafio ativo ou agendado; convites rastreáveis despachados. |
| **UC09** | Snapshot Diário e Indicadores Direcionais | Worker do Sistema | Desafio ativo; snapshot às 23:59:59 UTC [ESTABELECIDO]. | 1. Obtém ranking atual com desempate [ESTABELECIDO].<br>2. Copia `current_rank` para `previous_rank`.<br>3. Calcula setas comparativas para a interface. | Indicadores direcionais atualizados. |
| **UC10** | Síntese de Avatar Épico no Trono com IA | Worker de IA (GPU) | CHAMPION_THRONE: vencer no mínimo 5 desafios seguidos; quórum mínimo de 3 participantes [ESTABELECIDO]. | 1. Fecha o período de auditoria de 3 horas [ESTABELECIDO].<br>2. Confere autorização de uso de imagem [ESTABELECIDO].<br>3. Renderiza imagem do campeão com elementos previstos em `idea.md`.<br>4. Salva WebP no storage e atualiza `badges`.<br>5. Silhueta genérica para coadjuvante sem consentimento [ESTABELECIDO]. | Badge concedido; notificação e compartilhamento disponíveis. |
| **UC11** | Criar Campanha e Checkout Stripe | Anunciante / Personal | Cadastro de anunciante ativo com `tax_id`. | 1. Anunciante cadastra banner, esporte-alvo e raio PostGIS em km.<br>2. Backend cria sessão no Stripe Checkout.<br>3. Anunciante efetua pagamento no ambiente seguro Stripe.<br>4. Webhook `checkout.session.completed` ativa campanha (`active = true`). | Anúncio em rotação contextual no feed respeitando raio métrico e esporte. |

---

## 2. Tabelas de Decisão Executáveis

### 2.1 Tabela de Decisão 1: Processamento de Atividade Física

| Regra | Trajeto GPS Válido ($> 500\text{m}$) | Telemetria Cardíaca Presente | Duração em Movimento ($t$) | Bônus PR Strava | $F_{\text{ambiente}}$ | $F_{\text{FC}}$ | $F_{\text{consistência}}$ | Fórmula de Pontuação |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **R01** | Sim (Outdoor) | Sim | $60\text{ min}$ | 2 `best_efforts[].pr_rank=1`, cada um com melhoria $\ge5\%$ ($2\times25$) | $1.30$ | Calculado ($1.0\text{–}1.5$) | $1.05$ | $(\text{MET} \times 1.0 \times 1.30 \times F_{\text{FC}} \times 1.05 \times F_{\text{base}}) + 50.0$ |
| **R02** | Sim (Outdoor) | Não | $30\text{ min}$ | $0\text{ PR}$ | $1.30$ | $1.00$ | $1.00$ | $(\text{MET} \times 0.5 \times 1.30 \times 1.00 \times 1.00 \times F_{\text{base}}) + 0.0$ |
| **R03** | Não (Indoor) | Sim | $120\text{ min}$ | $0\text{ PR}$ | $1.00$ | Calculado ($1.0\text{–}1.5$) | $1.15$ | $(\text{MET} \times 2.0 \times 1.00 \times F_{\text{FC}} \times 1.15 \times F_{\text{base}}) + 0.0$ |
| **R04** | Não (Indoor) | Não | $40\text{ min}$ | $0\text{ PR}$ | $1.00$ | $1.00$ | $1.00$ | $(\text{MET} \times \frac{2}{3} \times 1.00 \times 1.00 \times 1.00 \times F_{\text{base}}) + 0.0$ |
| **R05** | Sim (Outdoor) | Sim | $180\text{ min}$ | $1\text{ KOM}$ ($50\text{ pts}$) | $1.30$ | Calculado ($1.0\text{–}1.5$) | $1.25$ | $(\text{MET} \times 3.0 \times 1.30 \times F_{\text{FC}} \times 1.25 \times F_{\text{base}}) + 50.0$ |

---

### 2.2 Tabela de Decisão 2: Bioimpedância e Pesagem

Para linhas B01–B09, $L$ e $G$ são os deltas efetivos após zerar **cada termo separadamente** quando $|\Delta|<0.2\text{ kg}$ [ESTABELECIDO]. Deltas exatamente $+0.2$ ou $-0.2$ não são zerados. A grade dos nove sinais é mutuamente exclusiva e exaustiva.

| Regra | Possui Bioimpedância? | $\Delta M_{\text{magra}}$ | $\Delta M_{\text{gorda}}$ | Intervalo $\ge 48\text{h}$? | Ação do Motor de Pontuação | Notificação / Diagnóstico |
| :---: | :---: | :---: | :---: | :---: | :--- | :--- |
| **B00** | Sim | Primeira pesagem | Primeira pesagem | N/A | $0.0$ (estabelece baseline) | Baseline registrado |
| **B01** | Sim | $L>0$ | $G<0$ | Sim | $150L-100G$ | Ganho magro e perda de gordura |
| **B02** | Sim | $L>0$ | $G=0$ | Sim | $150L$ | Ganho de massa magra |
| **B03** | Sim | $L>0$ | $G>0$ | Sim | $150L-100G$ | Saldo líquido; gordura reduz o bônus |
| **B04** | Sim | $L=0$ | $G<0$ | Sim | $-100G$ (positivo) | Perda de gordura com massa magra preservada |
| **B05** | Sim | $L=0$ | $G=0$ | Sim | $0.0$ | Variação dentro da banda de ruído |
| **B06** | Sim | $L=0$ | $G>0$ | Sim | $-100G$ | Ganho de gordura |
| **B07** | Sim | $L<0$ | $G<0$ | Sim | $150L-100G$ | Saldo líquido e alerta de perda de massa magra |
| **B08** | Sim | $L<0$ | $G=0$ | Sim | $150L$ (negativo) | Penalidade e alerta de perda de massa magra |
| **B09** | Sim | $L<0$ | $G>0$ | Sim | $150L-100G$ (negativo) | Penalidade e alerta de perda de massa magra |
| **B10** | Sim | Qualquer | Qualquer | Não | $0.0$ até atingir cooldown de 48h [ESTABELECIDO] | Medição permanece no histórico |
| **B11** | Não (balança comum) | N/A | N/A ($\Delta$ peso $=-0.8\text{ kg}$) | Sim | $+0.8\times40=+32$ [ESTABELECIDO] | Perda de peso dentro do teto |
| **B12** | Não (balança comum) | N/A | N/A ($\Delta$ peso $=-2.5\text{ kg}$ em até 7 dias) | Sim | Teto $+40$ [ESTABELECIDO] | Excesso não pontuado |
| **B13** | Não (balança comum) | N/A | N/A ($\Delta$ peso $\ge0$) | Sim | $0.0$ sem confirmação de ganho muscular | Manutenção ou aumento |

---

### 2.3 Tabela de Decisão 3: Auditoria de Sedentarismo

| Regra | Cenário de Avaliação | Tempo Inativo Decorrido | Duração da Atividade | Rest Day Prescrito? | Penalidade Aplicada | Justificativa |
| :---: | :--- | :---: | :---: | :---: | :---: | :--- |
| **S01** | Janela rolling de 24h | $<24.0\text{ h}$ desde o término anterior | Nova atividade qualificada | N/A | $0.0$ | Nenhum bloco completo inativo |
| **S02** | Janela rolling de 24h | $\ge24.0\text{ h}$ desde o término anterior | Nenhuma atividade qualificada | N/A | $-30.0$ (valor usado nos exemplos) | Penalidade da regra canônica |
| **S03** | Alternativa Dia Civil [ESTABELECIDO] | Dia civil encerrado | Sem treino | Rest Day adotado | $0.0$ | Isenção de descanso planejado |
| **S04** | Alternativa Dia Civil [ESTABELECIDO] | Dia civil encerrado | 10 min | Não | $-30.0$ | Abaixo do limiar de 15 min [ESTABELECIDO] |
| **S05** | Evento tardio | Prazo de penalidade já lançado, mas interrompido por atividade qualificada antes do prazo | Atividade reordenada por `start_date` | N/A | Estorno do lançamento devido, via conciliação idempotente | Não se aplica grace period civil |

---

## 3. Exemplos Numéricos Executáveis Ponta a Ponta

### Exemplo 1: Corrida de Rua Externa com Frequência Cardíaca e Recordes Pessoais
- **Perfil do Atleta**: Idade = 30 anos; $\text{FC}_{\text{repouso}} = 60\text{ bpm}$ e $\text{FC}_{\text{máx}}$ estimada por Tanaka [ESTABELECIDO]; $F_{\text{baseline}} = 1.00$ [ESTABELECIDO].
- **Atividade**: Corrida Outdoor de 60 minutos em movimento ($1.0\text{ h}$) a $10\text{ km/h}$; $\text{FC}_{\text{média}} = 160\text{ bpm}$; dois melhores esforços (`best_efforts`) com `pr_rank=1`, sem PR de segmento/KOM.
- **Dados de melhoria [ESTABELECIDO]**: esforço de 5 km: $1{,}800\to1{,}710$ s ($5\%$); esforço de 10 km: $3{,}790\to3{,}600$ s (mais de $5\%$). Cada esforço atinge o limite de bônus de $25.0$ pontos.
- **Passo a Passo com Arredondamento Canônico (`HALF_UP`, 2 casas)**:
  1. $\text{MET}_{\text{tipo}} = 9.8$ (Corrida moderada $10\text{ km/h}$).
  2. $F_{\text{ambiente}} = 1.30$ (Outdoor com trajeto GPS $> 500\text{m}$).
  3. $\text{FC}_{\text{máx}} = 208 - (0.7 \times 30) = 208 - 21 = 187\text{ bpm}$.
  4. $F_{\text{FC}} = 1.0 + \frac{160 - 60}{187 - 60} \times 0.50 = 1.0 + \frac{100}{127} \times 0.50 = 1.0 + (0.7874015 \times 0.50) = 1.3937007$.
  5. $F_{\text{consistência}} = 1.05$ (Duração entre 45 e 90 min).
  6. Para cada esforço, $r=\max(0,(t_{anterior}-t_{atual})/t_{anterior})$ e $B=25.0\times\min(1,r/0.05)$. Assim, $B_{5km}=25.00$ e $B_{10km}=25.00$; `pr_count=2` é apenas um indicador resumido e não é somado novamente. $B_{\text{troféus}}=+50.00\text{ pts}$.
  7. **Cálculo da Pontuação Bruta de Treino**:
     $$\text{Pontos}_{\text{brutos}} = 9.8 \times 1.0 \times 1.30 \times 1.3937007 \times 1.05 \times 1.00 = 18.643503... \approx \mathbf{18.64\text{ pts}}$$
  8. **Pontuação Total da Sessão**:
     $$\text{Total} = 18.64 + 50.00 = \mathbf{68.64\text{ pontos}}$$

---

### Exemplo 2: Recomposição Corporal na Bioimpedância
- **Situação**: O atleta realizou uma pesagem após 14 dias de treinos intensos.
- **Deltas Apurados**:
  - $\text{Massa Magra}: +0.80\text{ kg}$ ($\ge +0.2\text{kg}$)
  - $\text{Massa Gorda}: -1.50\text{ kg}$ ($\le -0.2\text{kg}$)
- **Cálculo da Pontuação**:
  $$\text{Pontos}_{\text{bio}} = (+0.80 \times 150.0) - (-1.50 \times 100.0) = 120.00 + 150.00 = \mathbf{+270.00\text{ pontos}}$$
- **Resultado**: $+270.00$ pontos adicionados diretamente à pontuação global acumulada ($P_{\text{global}}$).

---

### Exemplo 3: Dieta Restritiva Nociva com Perda de Massa Magra
- **Situação**: O atleta perdeu peso rápido cortando calorias drasticamente sem aporte proteico nem musculação.
- **Deltas Apurados**:
  - $\text{Peso Total}: -2.00\text{ kg}$
  - $\text{Massa Magra}: -1.20\text{ kg}$ (perda sarcopênica acentuada)
  - $\text{Massa Gorda}: -0.80\text{ kg}$
- **Cálculo da Pontuação**:
  $$\text{Pontos}_{\text{bio}} = (-1.20 \times 150.0) - (-0.80 \times 100.0) = -180.00 + 80.00 = \mathbf{-100.00\text{ pontos}}$$
- **Resultado**:
  - Dedução de $100.00$ pontos do score global acumulado ($P_{\text{global}}$).
  - Alerta médico/nutricional gerado pela IA recomendando aumento proteico e exercícios resistidos de força.

---

### Exemplo 4: Desafio em Grupo com Snapshot Diário e Inversão de Posição
- **Participantes**: Atleta A e Atleta B no *"Desafio 15 Dias"*.
- **Dia 1 (23:59:59 UTC - Snapshot)**:
  - Atleta A: $120.0\text{ pts}$ $\rightarrow$ `current_rank = 1`, `previous_rank = 1` ($\leftrightarrow$).
  - Atleta B: $95.0\text{ pts}$ $\rightarrow$ `current_rank = 2`, `previous_rank = 2` ($\leftrightarrow$).
- **Dia 2 (10:00 UTC - Atleta B realiza pedal longo outdoor)**:
  - Atleta B ganha $+45.0\text{ pts}$. Nova pontuação: $140.0\text{ pts}$.
  - Atleta A não treinou pela manhã: permanece com $120.0\text{ pts}$.
- **Reavaliação em Tempo Real**:
  - Atleta B: $140.0\text{ pts}$ $\rightarrow$ `current_rank = 1`. Como `previous_rank` era 2: **Seta $\uparrow$ (Subiu 1 posição)**.
  - Atleta A: $120.0\text{ pts}$ $\rightarrow$ `current_rank = 2`. Como `previous_rank` era 1: **Seta $\downarrow$ (Desceu 1 posição)**.

---

### Exemplo 5: Atividade sem Vestível (Sem Frequência Cardíaca)
- **Perfil do Atleta**: Idade = 35 anos; sem relógio esportivo ou monitor cardíaco; $F_{\text{baseline}} = 1.00$.
- **Atividade**: Caminhada rápida outdoor de 45 minutos em movimento ($0.75\text{ h}$), sem telemetria de FC.
- **Passo a Passo**:
  1. $\text{MET}_{\text{tipo}} = 3.8$ (Caminhada rápida $5\text{ km/h}$).
  2. $F_{\text{ambiente}} = 1.30$ (Outdoor com trajeto GPS $> 500\text{m}$).
  3. **Sem sensor cardíaco**: $F_{\text{FC}} = 1.00$ (regra canônica, sem penalidade).
  4. $F_{\text{consistência}} = 1.05$ (Duração entre 45 e 90 min).
  5. **Cálculo da Pontuação**:
     $$\text{Pontos} = 3.8 \times 0.75 \times 1.30 \times 1.00 \times 1.05 \times 1.00 = 3.9285 \approx \mathbf{3.93\text{ pontos}}$$
- **Resultado**: O atleta sem vestível recebe pontos calculados normalmente, sem penalidade. A ausência de FC apenas impede a bonificação cardiovascular ($F_{\text{FC}}$ fixo em $1.00$).

---

### Exemplo 6: Reconciliação de Inatividade sob a Regra Rolling de 24h
- **Situação [EXEMPLO DA REGRA CANÔNICA]**: A última atividade já conhecida terminou na segunda às 18:00 UTC. Na terça, o atleta faz uma corrida Outdoor de 40 min em movimento, das 17:00 às 17:40 UTC, mas o webhook só chega às 18:10. Sem o evento ainda conhecido, às 18:00 o job lança a penalidade ilustrativa de $-30.00$ pontos.
- **Pontuação da corrida**: sem FC, $F_{FC}=1.00$; MET $=9.8$; $F_{ambiente}=1.30$; $F_{consistência}=1.00$; $F_{baseline}=1.00$ (parâmetros assumidos). $9.8\times(40/60)\times1.30=8.4933\ldots\to+8.49$ pontos.
- **Reconciliação**:
  1. Às 18:10, o worker ordena a corrida por seu início real e calcula o término com `elapsed_time` (17:40 UTC), anterior ao prazo de penalidade (18:00 UTC).
  2. A corrida interrompe o bloco de 24h inativas; o sistema registra estorno compensatório de $+30.00$ pontos, ligado ao lançamento penalizado.
  3. Credita os $+8.49$ pontos de atividade e reinicia o contador após o término da corrida.
  4. O saldo final relativo ao evento é $+8.49$ pontos. Não há auditoria civil nem grace period contado da meia-noite.

---

## 4. Quadro Canônico de Resolução de Contradições e Decisões de Arquitetura

| Tópico Contraditório | Texto / Rascunho Inicial | Problema Prático Identificado | Resolução Canônica FitFight |
| :--- | :--- | :--- | :--- |
| **Pontuação de Desafio vs. Correções** | Requisito de crescimento de pontos durante o desafio versus `update`/`delete` de atividades. | A regra de acréscimo não pode proibir correções da fonte, que podem reduzir pontos. | **[ESTABELECIDO]**: nova atividade elegível soma; alterações, exclusões e revalidações podem reduzir o total recalculado. Bioimpedância/inatividade ficam fora do modo padrão; placar encerrado é congelado. |
| **Entrega de eventos Strava** | Rascunho supunha validação de assinatura no POST. | POSTs de evento não carregam assinatura criptográfica; o `GET` de handshake usa `hub.verify_token`/`hub.challenge`. | **[ESTABELECIDO]**: validar `subscription_id`, `owner_id` e callback secreto; enfileirar e confirmar em < 2s; buscar atividade autenticada via API. |
| **Dados de PR no webhook/API Strava** | *“quando o webhook sinalizar pr_count > 0”* e recordes por atividade/segmento/rota. | POST não contém detalhes; não há PR de rota no contrato usado; `pr_count` não deve ser somado aos detalhes. | **[ESTABELECIDO]**: buscar `ActivityDetail`; identificar best efforts e segmentos por seus campos `pr_rank`, e KOM/CR por `kom_rank`; PR+KOM somam; rotas não recebem PR por esse fluxo. |
| **Inatividade e atraso de sincronização** | Requisito de 24 horas consecutivas sem atividade. | Dia civil não representa uma janela móvel; atividade atrasada pode mudar a sequência de prazos. | **[ESTABELECIDO]**: contador rolling a partir do término calculado por `start_date + elapsed_time`; evento tardio reconcilia lançamentos e reverte somente prazos interrompidos. Dia Civil/Rest Day é alternativa adicional [ESTABELECIDO]. |
| **Escopo das insígnias** | `idea.md` prevê vencer todos os desafios do usuário. | A minuta convertia isso em vitória de um único desafio e fixava quórum/snapshot sem aprovação. | **[ESTABELECIDO]**: CHAMPION_THRONE requer vencer no mínimo 5 desafios seguidos; quórum mínimo de 3 participantes; snapshots e cooldowns definidos. |
