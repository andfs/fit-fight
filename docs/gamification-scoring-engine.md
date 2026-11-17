# Motor de Gamificação e Pontuação Normalizada (MET)

[Voltar ao Índice Geral](index.md)

---

## 1. Princípios e Taxonomia de Pontuação

O motor de pontuação do FitFight quantifica o esforço físico relativo e a composição corporal real dos participantes por meio de duas dimensões fundamentais:

1. **Pontuação Global Acumulada ($P_{\text{global}}$)** [ESTABELECIDO]:
   - Representa o histórico vitalício e a evolução contínua do atleta na plataforma.
   - Inicializada no cadastro com a pontuação de condicionamento físico inicial ($P_{\text{inicial}}$) calculada a partir dos dados de integração (conforme `idea.md`).
   - Sofre incrementos contínuos por atividades físicas registradas e evoluções saudáveis de bioimpedância.
   - Sofre deduções por sedentarismo (período sem atividade) e perda prejudicial de massa muscular magra.
   - Possui piso de proteção em $0.0$ ($P_{\text{global}} \ge 0.0$) para evitar desmotivação com saldos negativos [ESTABELECIDO].

2. **Pontuação Relativa em Desafios ($P_{\text{desafio}}$)** [ESTABELECIDO]:
   - Representa o score competitivo restrito à janela de vigência de um desafio específico $[t_{\text{início}}, t_{\text{fim}}]$.
   - Todos os participantes iniciam rigorosamente com $0.0$ pontos relativos no momento da largada ou do ingresso.
   - Pontua estritamente os treinos e atividades físicas elegíveis ocorridos dentro da vigência da competição.
   - No modo padrão, somente o acréscimo normal de novas atividades elegíveis é não-decrescente. Correções e reversões da fonte (atividade alterada, excluída ou revalidada) podem reduzir o score; o placar é recalculado a partir das atividades atualmente válidas. Bioimpedância e inatividade não alteram o placar padrão.

### 1.1 Pontos Iniciais de Fitness no Cadastro ($P_{\text{inicial}}$) [ESTABELECIDO]
`idea.md` estabelece que a pontuação inicial deve considerar dados das integrações. A fórmula canônica analisa os últimos 30 dias de treinos conectados:
  $$P_{\text{inicial}} = \min\left(200.0, (N_{\text{atividades\_30d}} \times 5.0) + (H_{\text{total\_30d}} \times 10.0)\right)$$
- **Cold start sem integração**: Se o usuário não integrar o Strava no cadastro, a pontuação inicial é $0.0$ (sem pontos de boas-vindas). O multiplicador $F_{\text{baseline}} = 1.00$ é aplicado até haver janela histórica suficiente.

---

## 2. Fórmula Matemática de Pontuação de Atividades

A pontuação concedida por uma atividade física concluída e validada é computada pela função canônica:

$$\text{Pontos}_{\text{atividade}} = \left( \text{MET}_{\text{tipo}} \times \frac{\text{Duração}_{\text{segundos}}}{3600} \times F_{\text{ambiente}} \times F_{\text{FC}} \times F_{\text{consistência}} \times F_{\text{baseline}} \right) + \text{Bônus}_{\text{troféus}}$$

### 2.1 Métrica de Duração e Timing da Atividade [ESTABELECIDO]
- **Duração Válida**: Utiliza-se estritamente o tempo em movimento (`moving_time` em segundos fornecido pela telemetria do Strava), e **não** o tempo bruto decorrido (`elapsed_time`), impedindo que pausas prolongadas (ex: paradas em semáforos, cafeterias ou descanso estático) inflem indevidamente a pontuação.
- **Data/Hora de Ocorrência (`occurred_at`)**: Representa o instante de início da atividade fornecido por `start_date` em UTC. `start_date_local` é horário de parede local para exibição e não deve ser interpretado diretamente como UTC.
- **Atividades que Cruzam Fronteiras Temporais**: Uma atividade cuja execução cruze a meia-noite civil ou a borda de início de um desafio é atribuída integralmente ao instante de seu início (`occurred_at`), garantindo integridade e atomicidade do evento esportivo.

### 2.2 Equivalente Metabólico da Tarefa ($\text{MET}_{\text{tipo}}$) [ESTABELECIDO]
O MET mensura o custo energético de uma atividade em múltiplos da Taxa Metabólica de Repouso ($1\text{ MET} \approx 1\text{ kcal/kg/h} \approx 3.5\text{ ml O}_2\text{/kg/min}$). O FitFight mapeia as atividades registradas ou importadas do Strava para a tabela oficial baseada no *Compendium of Physical Activities*:

| Modalidade Strava (`activity_type`) | Subtipo / Ritmo / Intensidade | $\text{MET}_{\text{tipo}}$ [ESTABELECIDO] | Descrição Fisiológica |
| :--- | :--- | :---: | :--- |
| `Run` (Corrida) | Ritmo Moderado ($10\text{ km/h}$ ou $6\text{ min/km}$) | $9.8$ | Corrida contínua padrão |
| `Run` (Corrida) | Ritmo Forte ($\ge 12\text{ km/h}$ ou $\le 5\text{ min/km}$) | $11.5$ | Corrida de alta intensidade |
| `Ride` (Ciclismo) | Ciclismo Recreativo / Urbano ($16\text{–}19\text{ km/h}$) | $6.8$ | Esforço moderado |
| `Ride` (Ciclismo) | Ciclismo de Estrada ($20\text{–}25\text{ km/h}$) | $8.5$ | Esforço vigoroso |
| `Ride` (Ciclismo) | Ciclismo Rápido / Contra-relógio ($> 25\text{ km/h}$) | $10.5$ | Esforço muito vigoroso |
| `Swim` (Natação) | Nado Livre Moderado | $7.0$ | Esforço contínuo aeróbico |
| `Swim` (Natação) | Nado Rápido / Crawl Intenso | $10.0$ | Esforço vigoroso |
| `WeightTraining` | Treino de Força / Hipertrofia Geral | $5.0$ | Musculação com pausas |
| `Crossfit` / `Workout` | Treino Funcional de Alta Intensidade | $8.0$ | Circuitos e intervalos curtos |
| `Walk` (Caminhada) | Caminhada Rápida ($5\text{ km/h}$) | $3.8$ | Deslocamento ativo |
| `Hike` (Trilha) | Caminhada em Declive / Terreno Irregular | $6.0$ | Carga mecânica adicional |
| `Yoga` / `Pilates` | Postural e Flexibilidade | $3.0$ | Atividade de baixo impacto |

*Fallback [ESTABELECIDO]*: caso o tipo de atividade não conste na tabela, aplica-se $\text{MET} = 5.0$ genérico.

### 2.3 Fator de Ambiente ($F_{\text{ambiente}}$) [ESTABELECIDO]
Conforme especificado em `idea.md`, treinos externos expõem o atleta a atrito aerodinâmico, irregularidades do piso, declives e variações climáticas, devendo ser melhor recompensados:

$$F_{\text{ambiente}} = \begin{cases} 1.30, & \text{se a atividade for externa (outdoor: corrida de rua, ciclismo em via, águas abertas)} \\ 1.00, & \text{se a atividade for interna (indoor: esteira, rolo de bike, natação em piscina, academia)} \end{cases}$$

A detecção de atividade externa ocorre via telemetria: presença de trajeto GPS válido com mais de $500\text{ metros}$ de deslocamento [ESTABELECIDO] e sinalização `trainer = false` na API do Strava.

### 2.4 Fator de Frequência Cardíaca ($F_{\text{FC}}$) [ESTABELECIDO]
A resposta cardíaca valida a intensidade fisiológica do esforço, mitigando trapaças como registro passivo de deslocamento motorizado:

$$F_{\text{FC}} = 1.0 + \max\left(0.0, \frac{\text{FC}_{\text{média}} - \text{FC}_{\text{repouso}}}{\text{FC}_{\text{máx}} - \text{FC}_{\text{repouso}}}\right) \times 0.50$$

- $\text{FC}_{\text{máx}}$: Frequência Cardíaca Máxima do atleta. A fórmula de Tanaka é o padrão canônico [ESTABELECIDO]:
  $$\text{FC}_{\text{máx}} = 208 - (0.7 \times \text{Idade})$$
- $\text{FC}_{\text{repouso}}$: Frequência Cardíaca de Repouso. Caso não informada nos dados do Strava ou nos vestíveis, adota-se o fallback de $60\text{ bpm}$ [ESTABELECIDO].
- **Cenário sem sensor cardíaco**: Se a atividade não possuir telemetria de batimentos cardíacos (`has_heartrate = false`), o fator é fixado em $F_{\text{FC}} = 1.00$. O atleta não é penalizado, mas não aufere a bonificação de esforço cardiovascular.
- **Teto do Fator de FC**: $1.00 \le F_{\text{FC}} \le 1.50$.

### 2.5 Fator de Consistência e Duração Longa ($F_{\text{consistência}}$) [ESTABELECIDO]
Em conformidade com a premissa de que *"um treino externo longo deve pontuar mais que um externo curto, e mais que um interno longo"*, atividades que superam limiares de fadiga recebem um multiplicador progressivo de consistência:

$$F_{\text{consistência}} = \begin{cases} 1.00, & \text{se } \text{Duração} < 45\text{ minutos} \\ 1.05, & \text{se } 45\text{ min} \le \text{Duração} < 90\text{ minutos} \\ 1.15, & \text{se } 90\text{ min} \le \text{Duração} < 150\text{ minutos} \\ 1.25, & \text{se } \text{Duração} \ge 150\text{ minutos (ultra-endurance)} \end{cases} \quad [\text{ESTABELECIDO}]$$

A combinação $F_{\text{ambiente}} \times F_{\text{consistência}}$ assegura a hierarquia matemática estrita:
$$\text{Outdoor Longo} (1.30 \times 1.25 = 1.625) > \text{Outdoor Curto} (1.30 \times 1.00 = 1.300) > \text{Indoor Longo} (1.00 \times 1.25 = 1.250)$$

### 2.6 Calibração pelo Baseline de Condicionamento ($F_{\text{baseline}}$) [ESTABELECIDO]
O sistema calcula o volume médio semanal ($V_{\text{semanal}}$ em horas) em janela móvel dos últimos 30 dias para calibrar a pontuação:

1. **Faixas de Volume e Baseline** [ESTABELECIDO]:
   $$F_{\text{baseline}} = \begin{cases} 1.20, & \text{Iniciante: } V_{\text{semanal}} < 2.0\text{ horas/semana} \\ 1.00, & \text{Intermediário: } 2.0\text{ horas/semana} \le V_{\text{semanal}} \le 6.0\text{ horas/semana} \\ 0.90, & \text{Avançado / Elite: } V_{\text{semanal}} > 6.0\text{ horas/semana} \end{cases}$$

2. **Usuários sem Histórico Prévio (Cold Start)** [ESTABELECIDO]:
   Atribui-se $F_{\text{baseline}} = 1.00$ até haver janela histórica suficiente; isso não implica bônus ou penalidade de iniciante.

3. **Recalibração Contínua e Prevenção de Fraude / Gaming** [ESTABELECIDO]:
   Para evitar que um atleta experiente oculte intencionalmente seu histórico para obter o bônus de iniciante ($1.20$), o $F_{\text{baseline}}$ é **recalculado automaticamente a cada 30 dias** com base no volume real executado. Se um usuário registrar alto volume nas primeiras 4 semanas, seu baseline é promovido no ciclo seguinte para $1.00$ ou $0.90$.

4. **Taxa Metabólica Basal (TMB)** [NÃO DEFINIDO]:
   A fórmula principal deste documento **não contém TMB** e não há coeficiente definido para usá-la como fator de pontuação. Portanto, a TMB não deve alterar pontos. Mifflin-St Jeor pode ser usada futuramente para estimativa fisiológica em insights de saúde, separada do cálculo de score:
   $$\text{TMB}_{\text{masculino}} = (10 \times \text{peso}_{\text{kg}}) + (6.25 \times \text{altura}_{\text{cm}}) - (5 \times \text{idade}) + 5$$
   $$\text{TMB}_{\text{feminino}} = (10 \times \text{peso}_{\text{kg}}) + (6.25 \times \text{altura}_{\text{cm}}) - (5 \times \text{idade}) - 161$$

---

## 3. Ingestão do Strava, Ciclo de Vida e Bonificação de Recordes (PRs)

### 3.1 Ciclo de Vida do Webhook do Strava (`create`, `update`, `delete`, `deauthorize`) [ESTABELECIDO]
O webhook do Strava opera por handshake de inscrição e entrega assíncrona via PostgreSQL PGMQ:
1. **Verificação da inscrição**: no `GET` de callback, o backend verifica `hub.mode = subscribe`, compara `hub.verify_token` com o segredo configurado e responde com `hub.challenge`. Essa verificação ocorre na criação/validação da inscrição, não assina os eventos.
2. **Recepção do evento**: os `POST` de evento do Strava **não incluem assinatura criptográfica**. Validar que `subscription_id` corresponde à inscrição esperada e que `owner_id` corresponde a uma conta Strava vinculada; um caminho de callback secreto adicional é uma defesa obrigatória [ESTABELECIDO]. Enfileirar o evento e responder `HTTP 200 OK` rapidamente [meta de menos de 2 s: ESTABELECIDO].
3. **Consumo Assíncrono pelo Worker**:
   - Evento `aspect_type == 'create'`: O worker chama `GET /activities/{object_id}` com o token OAuth2 do usuário, calcula os pontos MET e bônus, persiste em `activities` e atualiza os placares.
   - Evento `aspect_type == 'update'`: sempre buscar novamente `GET /activities/{object_id}` e reavaliar idempotentemente. O campo `updates` da notificação pode conter `title`, `type` e `private`; não informa alterações de `gear_id` ou `distance`, portanto não usar o payload como filtro para decidir se recalcula.
   - Evento `aspect_type == 'delete'`: se uma atividade for excluída no Strava, o FitFight remove a atividade de `activities` e subtrai os pontos correspondentes de $P_{\text{global}}$ com registro de estorno em `points_history`. Em desafios `ACTIVE` ou `AUDIT_GRACE`, o placar é recalculado; após `COMPLETED`, o resultado histórico permanece congelado.
   - `private = true`: a leitura de atividade privada exige o escopo OAuth `activity:read_all`. Sem ele, marcar a atividade como não verificável e solicitar nova autorização; não interpretar privacidade como exclusão. Manter pontuação previamente verificada enquanto aguarda reautorização [ESTABELECIDO].
   - Desautorização: no evento de atleta `aspect_type = 'update'` com `updates.authorized = false`, revogar tokens locais e marcar a integração como desativada.

### 3.2 Bonificação por Personal Records (PRs): Intenção de Melhoria vs. API do Strava

#### Intenção Original em `idea.md` e Desafios de Integração
`idea.md` especifica que recordes pessoais em atividades, segmentos e rotas devem ser pontuados *"com base na melhoria em relação ao recorde anterior"* (proporcional à redução de tempo). Contudo, a integração real com a API pública v3 do Strava impõe duas restrições técnicas:
1. **Inexistência de PRs em Rotas**: a API pública do Strava não fornece um tipo de recorde pessoal por rota que possa ser pontuado por este fluxo. Não conceder bônus de rota com base apenas no webhook/API Strava.
2. **Campos por categoria**: `ActivityDetail.pr_count` é um total/sinalizador, não deve ser somado novamente aos detalhes. Identificar recordes de melhores esforços em distâncias padrão por `best_efforts[].pr_rank == 1`, recordes de segmento por `segment_efforts[].pr_rank == 1` e KOM/CR por `segment_efforts[].kom_rank == 1`. Um mesmo esforço de segmento que seja PR e KOM/CR simultaneamente **soma os dois bônus** [ESTABELECIDO].
3. **Tempo anterior**: o evento não traz a melhoria anterior como delta. Comparar `moving_time` do esforço atual com o melhor tempo anterior persistido por atleta e chave de esforço/segmento. Sem baseline anterior, registrar/estabelecer baseline sem bônus de melhoria [ESTABELECIDO]. Uma tabela dedicada `athlete_segment_records` é obrigatória para armazenar o histórico de recordes por segmento [ESTABELECIDO].

#### Modelo de Bonificação Canônico [ESTABELECIDO]
O modelo preserva a intenção de `idea.md` de premiar melhoria relativa, usando os campos detalhados acima:

$$r = \max\left(0, \frac{t_{\text{anterior}} - t_{\text{atual}}}{t_{\text{anterior}}}\right), \quad B_{\text{tipo}} = W_{\text{tipo}} \times \min\left(1, \frac{r}{0.05}\right)$$

Onde $W_{\text{best-effort}} = 25.0$, $W_{\text{segmento}} = 15.0$ e $W_{\text{KOM/CR}} = 50.0$ pontos máximos por conquista [ESTABELECIDO]. KOM/CR usa `segment_efforts[].kom_rank == 1`; não é calculado como melhoria percentual. `pr_count` pode servir como conferência do retorno detalhado, nunca como um bônus adicional.

- **Teto cumulativo de $150.0$ pontos por atividade** [ESTABELECIDO]: limite anti-abuso.

---

## 4. Motor de Bioimpedância e Recomposição Corporal

O módulo biométrico quantifica a melhora da composição corporal, premiando hipertrofia e queima de gordura e penalizando estratégias nocivas à saúde (sarcopenia e dietas restritivas extremas).

### 4.1 Derivação de Grandezas a partir do Esquema de Dados [ESTABELECIDO]
A tabela `user_biometrics` armazena `weight_kg`, `muscle_mass_kg`, `body_fat_percentage` e `water_percentage`. O motor deriva as variáveis de cálculo por:
- $\text{MassaGorda}_{\text{kg}} = \text{weight\_kg} \times \left( \frac{\text{body\_fat\_percentage}}{100.0} \right)$
- $\text{MassaMagra}_{\text{kg}} = \text{weight\_kg} - \text{MassaGorda}_{\text{kg}}$ (Massa Livre de Gordura / *Fat-Free Mass*)
- Quando disponível via balança de 8 eletrodos, `muscle_mass_kg` quantifica a massa muscular esquelética. Por padrão, a fórmula utiliza a massa magra total derivada ($\text{MassaMagra}_{\text{kg}}$).

### 4.2 Fórmula de Evolução Biométrica [ESTABELECIDO]
Para cada nova pesagem em relação à última medição validada:
$$\Delta \text{MassaMagra}_{\text{kg}} = \text{MassaMagra}_{\text{atual}} - \text{MassaMagra}_{\text{anterior}}$$
$$\Delta \text{MassaGorda}_{\text{kg}} = \text{MassaGorda}_{\text{atual}} - \text{MassaGorda}_{\text{anterior}}$$

$$\text{Pontos}_{\text{bio}} = (\Delta \text{MassaMagra}_{\text{kg}} \times K_{\text{magra}}) - (\Delta \text{MassaGorda}_{\text{kg}} \times K_{\text{gorda}})$$

Constantes adotadas conforme o plano:
- $K_{\text{magra}} = 150.0\text{ pts/kg}$ [ESTABELECIDO] (bonifica ganho muscular ou subtrai pontos se o delta efetivo após banda de ruído for negativo).
- $K_{\text{gorda}} = 100.0\text{ pts/kg}$ [ESTABELECIDO] (bonifica perda de gordura quando $\Delta M_{\text{gorda}} < 0$, pois $- (\text{negativo}) = \text{positivo}$).

### 4.3 Partição Completa e Mutuamente Exclusiva de Estados Biométricos [ESTABELECIDO]

Aplicar a banda de ruído **separadamente a cada termo**. Para tolerância $\epsilon = 0.2\text{ kg}$ [ESTABELECIDO], defina $\widehat{\Delta X}=0$ se $|\Delta X|<\epsilon$; caso contrário $\widehat{\Delta X}=\Delta X$. Assim, exatamente $+0.2$ e $-0.2$ kg não são zerados. Defina $L=\widehat{\Delta M}_{magra}$ e $G=\widehat{\Delta M}_{gorda}$; usar $\text{Pontos}_{bio}=150L-100G$. Um delta pequeno de gordura não apaga o termo de massa magra nem vice-versa.

| $\widehat{\Delta M}_{magra}$ | $\widehat{\Delta M}_{gorda}$ | Resultado | Interpretação / Alerta |
| :---: | :---: | :--- | :--- |
| $>0$ | $<0$ | $150L-100G$ (saldo positivo) | Ganho magro e perda de gordura |
| $>0$ | $=0$ | $150L$ (positivo) | Ganho de massa magra |
| $>0$ | $>0$ | $150L-100G$ | Saldo líquido; ganho de gordura reduz o bônus |
| $=0$ | $<0$ | $-100G$ (positivo) | Perda de gordura com massa magra preservada |
| $=0$ | $=0$ | $0$ | Estabilidade dentro da banda de ruído |
| $=0$ | $>0$ | $-100G$ (negativo) | Ganho de gordura |
| $<0$ | $<0$ | $150L-100G$ | Saldo líquido e alerta de perda de massa magra |
| $<0$ | $=0$ | $150L$ (negativo) | Penalidade e alerta de perda de massa magra |
| $<0$ | $>0$ | $150L-100G$ (negativo) | Penalidade e alerta de perda de massa magra |

O caso sem registro anterior é baseline e gera zero pontos [ESTABELECIDO]. Para qualquer delta efetivo $L<0$, o saldo líquido pode ser atenuado por perda de gordura, mas o alerta sobre perda de massa magra permanece; não substituir o cálculo pela categoria do quadro.

### 4.4 Balança Convencional (Sem Bioimpedância) e Ganho de Peso [ESTABELECIDO]
Quando o atleta registra apenas o peso corporal na balança comum:
- Se $\Delta \text{Peso}_{\text{kg}} < 0$ (perda de peso):
  - Limite admitido como saudável: **máximo de $1.0\text{ kg}$ por período de 7 dias** [ESTABELECIDO].
  - Pontuação concedida: $\text{Pontos}_{\text{peso}} = |\Delta \text{Peso}_{\text{kg}}| \times 40.0\text{ pts/kg}$ [ESTABELECIDO].
  - **Teto Semanal Anti-Fraude**: Pontuação limitada a $+40.0\text{ pontos}$ por semana. Qualquer perda excedente a $1.0\text{ kg}$ em 7 dias não gera pontos adicionais imediatos.
- Se $\Delta \text{Peso}_{\text{kg}} \ge 0$ (ganho de peso ou estabilidade): Sem confirmação de ganho muscular por bioimpedância, o aumento de peso **não concede pontos** ($0.0\text{ pontos}$).

### 4.5 Regras Operacionais de Cooldown e Ruído [ESTABELECIDO]
- **Cooldown entre Pesagens**: Intervalo mínimo de **48 horas** entre pesagens elegíveis a pontuação. Aferições intermediárias são salvas no gráfico histórico, mas não geram transações de pontos. **O frontend deve informar ao usuário este cooldown antes de registrar uma nova pesagem.**
- **Banda de Tolerância a Ruído Instrumental**: Variações de peso e massa magra inferiores a $\pm 0.2\text{ kg}$ são tratadas como oscilação hídrica neutra ($0.0\text{ pontos}$).

---

## 5. Penalidade por Inatividade (Sedentarismo)

### 5.1 Regra Canônica Base: 24 Horas Consecutivas sem Atividade [ESTABELECIDO]
O plano de requisitos canônico estabelece:
> *"Caso o usuário passe 24 horas consecutivas sem nenhuma atividade física registrada, uma penalidade fixa de inatividade é subtraída de sua pontuação global acumulada."*
E `idea.md` corrobora: *"If a user doesn't make any activity in a day, he will lose some points."*

**Definição Operacional da Regra Canônica**:
- Para o contador, usar o término da última atividade qualificada: $t_{fim}=A.\text{start\_date}+A.\text{elapsed\_time}$. `elapsed_time` inclui pausas e representa o intervalo decorrido; `moving_time` é usado para calcular pontos e não para adiantar o relógio de inatividade. `start_date` é UTC.
- A cada bloco completo de 24 horas sem atividade qualificada após $t_{fim}$, aplicar a penalidade fixa de $-30.0$ pontos [ESTABELECIDO]. Uma atividade qualificada reinicia o contador ao terminar. O limiar mínimo de 15 minutos em movimento é obrigatório [ESTABELECIDO] (§5.3).
- Se não houver atividade qualificada anterior, usar a data de cadastro como início do primeiro contador [ESTABELECIDO]. Registrar cada penalidade com o instante-limite e uma chave idempotente.

### 5.2 Abordagem Alternativa: Dia Civil Local e Rest Day [ESTABELECIDO]
Além da regra canônica rolling-24h, o sistema suporta uma alternativa por Dia Civil Local:
- **Motivação**: Em uma janela deslizante contínua de 24 horas, um atleta que treina às 07:00 da segunda-feira e às 18:00 da terça-feira sofreria penalidade (gap de 35 horas), embora tenha treinado nos dois dias úteis.
- **Mecânica**:
  1. A auditoria opera no encerramento do dia civil local do atleta (`00:00:00` às `23:59:59` no fuso horário do usuário).
  2. Um job noturno às `00:05` local audita o dia anterior. Se não houver treino qualificado, deduz $-30.0\text{ pontos}$.
  3. **Isenção de Descanso Planejado (Rest Day)**: Qualquer usuário tem direito a **1 dia de descanso planejado por semana** (configurado pelo atleta ou sugerido pelo Assistente IA), ficando isento da penalidade naquele dia civil para permitir supercompensação muscular fisiológica.

### 5.3 Limiar Qualificador de Treino (Gating Anti-Penalidade) [ESTABELECIDO]
Para quebrar o período de inatividade e evitar a penalidade diária:
- Exige-se que a atividade seja validada com **duração mínima de 15 minutos** (tempo em movimento).
- *Alerta de Correção*: Não se exige pontuação arbitrária elevada (ex: 15 pontos), pois caminhadas ou treinos leves de 15 a 30 minutos geram entre $1.0$ e $3.0\text{ pontos}$ e constituem atividade física saudável suficiente para romper o sedentarismo.

### 5.4 Sincronização Tardia e Reconciliação do Contador de 24h [ESTABELECIDO]
- Eventos tardios são ordenados pelo instante real da atividade, não pelo instante de recepção. Ao inserir, atualizar ou excluir uma atividade qualificada, recalcular a sequência de intervalos inativos a partir do primeiro instante afetado até agora.
- Comparar penalidades esperadas com `points_history`: reverter, com lançamento compensatório idempotente, cada penalidade cujo prazo foi interrompido por atividade qualificada; manter as penalidades correspondentes a blocos completos de 24h que continuam sem atividade. Se o novo histórico criar outro bloco completo, aplicá-lo.
- Não inferir o último evento apenas pelo maior/menor `start_date`: após inserção, atualização ou exclusão, reconstruir cronologicamente os intervalos usando cada `start_date` e `elapsed_time`; o contador volta a partir do término válido mais recente. Isso também trata sessões sobrepostas sem perder períodos de inatividade anteriores que tenham sido interrompidos pelo evento tardio.
- Não há prazo civil de 24h para aceitar a correção: a janela limitada a 24h após meia-noite pertence somente à alternativa Dia Civil [ESTABELECIDO].

---

## 6. Alerta de Desbalanceamento de Escala [ESTABELECIDO]

É imperativo registrar formalmente que a escala matemática combinada apresenta disparidades de magnitude que requerem calibração empírica antes do lançamento:

| Dimensão de Pontuação | Faixa Típica por Ocorrência | Frequência Esperada | Impacto no Placar |
| :--- | :---: | :---: | :--- |
| **Sessão Individual de Treino** | $+2.0\text{ a }+20.0\text{ pts}$ | Diária (1 a 2x/dia) | Motor primário contínuo de pontuação |
| **Penalidade Diária de Sedentarismo** | **$-30.0\text{ pts}$** | 1x por dia inativo | Equivale a perder de 2 a 3 sessões médias de treino |
| **Bônus de Troféus / PRs** | **$+25.0\text{ a }+50.0\text{ pts}$** | Ocasional (1 a 3x/mês) | Salto de pontuação comparável a 2 a 5 treinos inteiros |
| **Delta de Bioimpedância** | **$\pm 100.0\text{ a }\pm 270.0\text{ pts}$** | Quinzenal / Mensal | Pode superar todo o esforço de semanas de treino |

**Recomendações para Calibração de Produto**:
1. O texto original sugeria que "$-30\text{ pts}$ equivale a 30 min de caminhada". Na realidade do MET, 30 min de caminhada equivalem a apenas $\approx 1.9\text{ a }2.5\text{ pontos}$. Uma penalidade de $-30.0\text{ pts}$ é severa e deve ser avaliada contra a taxa de retenção de usuários novatos.
2. Os coeficientes de bioimpedância ($K_{\text{magra}} = 150.0$, $K_{\text{gorda}} = 100.0$) devem ser mantidos estritamente em pesagens espaçadas (mínimo 14 a 30 dias) ou reescalonados por um fator de $0.1$ ($K_{\text{magra}} = 15.0$, $K_{\text{gorda}} = 10.0$) caso pesagens frequentes comecem a distorcer a hierarquia dos atletas mais ativos.

---

## 7. Quadro Consolidado

| Elemento / Regra | Classificação | Valor Canônico / Detalhe | Justificativa / Origem |
| :--- | :---: | :---: | :--- |
| **Pontos Iniciais de Fitness** | [ESTABELECIDO] | $P_{\text{inicial}} = \min(200.0, (N_{30d} \times 5) + (H_{30d} \times 10))$; sem integração: $0.0$ | Fórmula canônica sem pontos de boas-vindas |
| **Score Relativo de Desafio zerado** | [ESTABELECIDO] | $P_{\text{desafio}} = 0.0$ no início da competição | Isonomia competitiva |
| **Fator Ambiente Outdoor vs. Indoor** | [ESTABELECIDO] | $1.30\text{ outdoor} \text{ vs } 1.00\text{ indoor}$ | Recompensa adversidades climáticas e atrito |
| **Duração por Tempo em Movimento** | [ESTABELECIDO] | `moving_time` em segundos | Elimina distorções de pausas |
| **Penalidade por perda de massa magra** | [ESTABELECIDO] | Subtração proporcional quando o delta efetivo de massa magra é negativo | Desencoraja dietas extremas prejudiciais |
| **Penalidade de 24h de Inatividade** | [ESTABELECIDO] | Janela de 24 horas consecutivas; $-30.0$ pontos por bloco completo | Regra formal e cenário de teste do Plano |
| **Alternativa Dia Civil e Rest Day** | [ESTABELECIDO] | 00:00 às 23:59 local + 1 Rest Day/semana | Alternativa adicional à regra rolling-24h |
| **Limiar para quebrar inatividade** | [ESTABELECIDO] | Duração $\ge 15\text{ minutos}$ | Evita bloqueio por nota de corte impossível |
| **Tabela de Coeficientes MET** | [ESTABELECIDO] | 12 modalidades mapeadas; fallback $\text{MET} = 5.0$ | Compendium of Physical Activities |
| **Coeficientes de Bioimpedância** | [ESTABELECIDO] | $K_{\text{magra}} = 150.0$, $K_{\text{gorda}} = 100.0$ | Coeficientes canônicos aprovados |
| **Bônus de PR por melhoria** | [ESTABELECIDO] | Pesos máximos de $25.0$ (best effort), $15.0$ (segmento) e $50.0$ (KOM/CR); escala de melhoria de 5%; PR+KOM somam | Preserva a intenção de `idea.md`; parâmetros aprovados |
| **Teto de bônus por atividade** | [ESTABELECIDO] | $150.0$ pontos cumulativos | Limite anti-abuso |
| **Banda de Tolerância de Ruído Bio** | [ESTABELECIDO] | $\pm 0.2\text{ kg}$ | Tolerância contra ruído instrumental |
| **Cooldown entre Pesagens** | [ESTABELECIDO] | $48\text{ horas}$ | Prevenção de oscilações hídricas; informar no frontend |
| **Reconciliação de Sincronização Tardia** | [ESTABELECIDO] | Recalcular períodos afetados e reverter somente prazos de inatividade interrompidos por atividade elegível | Preserva a regra canônica rolling-24h |
| **Taxa Metabólica Basal no Score** | [NÃO DEFINIDO] | Não participa da fórmula | Reservada para insights de saúde futuros |
| **FC de repouso fallback** | [ESTABELECIDO] | $60\text{ bpm}$ quando não informada | Fallback canônico |
| **Fórmula FC máxima Tanaka** | [ESTABELECIDO] | $\text{FC}_{\text{máx}} = 208 - (0.7 \times \text{Idade})$ | Fórmula canônica |
| **Detecção outdoor 500m** | [ESTABELECIDO] | GPS válido com $>500\text{m}$ + `trainer = false` | Limiar canônico |
| **Callback secreto Strava** | [ESTABELECIDO] | Caminho de callback secreto adicional obrigatório | Defesa de segurança |
| **Tabela `athlete_segment_records`** | [ESTABELECIDO] | Tabela dedicada para histórico de recordes por segmento | Armazenamento de baselines |
