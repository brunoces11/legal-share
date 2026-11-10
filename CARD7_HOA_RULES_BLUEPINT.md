# LEEVLI | LEGAL ANALYSIS | CARD 7 — HOA RULES AND OBLIGATIONS
# MASTER IMPLEMENTATION BLUEPRINT V2 — BASEADO NO TEMPLATE CARD6_HOA_FIN_BLUEPRINT.md (V3)
# (aderente ao código em 2026-11-06, incluindo o Plano A: coordenador de consultas por execução)

Este documento é o blueprint de implementação do Card 7 (HOA Rules and Obligations).
Segue a estrutura do template do Card 6: as seções [REGRA FIXA] são idênticas às do template;
as seções [ESPECÍFICO DO CARD] são próprias do Card 7.

V2 (2026-11-06): identidade própria sem coordenadas; status do worker pela regra 4.1; o Card 7
não determina regime nem usa DBPR; base legal pelo registro único de estatutos.

---------------------------------------------------------------------------------------------
## ⚠️ MUITO IMPORTANTE — LEIA ANTES DE QUALQUER LINHA DE CÓDIGO [REGRA FIXA]
---------------------------------------------------------------------------------------------
1. O QUE É COMUM A TODOS OS CARDS E O QUE É DE CADA CARD.
   Comum e obrigatório — o ENCAIXE do card no relatório:
   a) o encaixe no pipeline macro (seção 2), pelos pontos de integração (seção 5);
   b) os mecanismos compartilhados (seção 3): cesta, coordenador, SharedMlsQueries,
      registro de estatutos, options, log;
   c) o contrato do worker (seção 4.1): independência, tempo, isolamento de fontes, payload
      versionado e significado do status;
   d) o padrão de layout: acordeão expandido/compacto, STYLE GUIDE CARDS e os cards já
      implementados como referência visual.
   De cada card — NÃO se copia de outro card: fontes, campos, identidade de que precisa,
   hierarquia e prioridade dos dados, regras de negócio, estados e textos.
2. CARDS JÁ IMPLEMENTADOS SÃO REFERÊNCIA, NÃO MODELO. O Card 5 — FLOOD AND ENVIRONMENTAL
   RISKS (worker "flood-environmental-risk") é a referência de boas práticas e de código que
   funciona para problemas parecidos (seção 4.2). Reaproveite classes e funções existentes
   quando servem de fato ao card novo; nunca copie um comportamento de outro card só porque
   ele existe. O worker property-data-cache (cards 1, 3 e 4) é uma exceção histórica e não é
   referência.
3. REAPROVEITE OS MECANISMOS INTERNOS JÁ IMPLEMENTADOS (seção 3). Não crie cache, loop, retry,
   fila, orquestração, cliente ou registro paralelos ao que já existe.
4. NUNCA DUPLIQUE DADOS DE OUTROS CARDS. Um dado de outro card só pode aparecer como
   indicador ou referência (ex.: "ver Card 5"), nunca como dado de exposição repetido.
   ANTES de definir que um dado será exibido no card final do relatório, verifique se ele já
   é exibido por outro card deste mesmo relatório (LISTA_CARDS.txt, PROMPT_DEV_LEGAL.txt,
   blueprints dos cards já especificados, mappers e componentes em
   features/listings/data-access e ui/legal-analysis/cards).
   Reutilizar o MESMO dado em cache para cálculo ou identidade é permitido e desejado;
   exibi-lo de novo não é.
5. ATUE COMO DESENVOLVEDOR SÊNIOR, ALTAMENTE PRAGMÁTICO E EFICIENTE. Garanta que não haverá
   nenhuma quebra no fluxo atual: cards já implementados, Summary, LLM, persistência
   (Building, PropertyReport, Blob), contratos HTTP e testes existentes continuam idênticos.
6. O CARD É ACOPLÁVEL. O codebase já foi projetado para receber novos cards: o card entra
   pelos pontos de integração listados na seção 5, sem alterar a semântica de nenhum outro
   card, worker, contrato ou fluxo. Se algo exigir mudar outro componente, pare e reporte ao
   arquiteto antes.

---------------------------------------------------------------------------------------------
## 1. NOMENCLATURA OFICIAL [REGRA FIXA]
---------------------------------------------------------------------------------------------
- Card ........ seção apresentada no relatório (frontend), numerada pelo LISTA_CARDS.txt.
- Worker ...... unidade do backend que coleta e organiza os dados de UM card.
                Regra: 1 card = 1 worker; Id do worker = slug do card.
- Fonte ....... consulta ou serviço externo usado por um worker (API, base pública).
- Fonte de conteúdo  fonte que entrega os dados que o card exibe; define o status (4.1).
- Pré-requisito  dado de identidade de que o card precisa para saber se se aplica ao imóvel
                (ex.: estado, coordenadas). Não é dado exibido nem fonte de conteúdo.
- Cesta ....... cache existente (IMemoryCache nos clientes de integração, lido pelo
                IPropertyDataCacheReader).
- Coordenador . peça do Plano A que evita consultas simultâneas iguais na mesma execução.
- Registro de estatutos  lista interna única de Florida Statutes usada pelos cards (3.4).
- Consolidação  etapa após os workers: agrega os resultados e chama o LLM (Summary/Card 2).
- Exceção documentada: o worker property-data-cache alimenta os cards 1, 3 e 4. Será migrado
  para o padrão no futuro; não será dividido agora.

---------------------------------------------------------------------------------------------
## 2. PIPELINE E PONTOS DE INTEGRAÇÃO EXISTENTES (NÃO ALTERAR SEMÂNTICA) [REGRA FIXA]
---------------------------------------------------------------------------------------------
- Fluxo: NewLegalAnalysisController -> RunLegalAnalysisHandler -> ILegalAnalysisOrchestrator
  (workers em paralelo) -> LegalAnalysisReportAttemptService (LLM + validação) -> Building
  -> Blob -> PropertyReport.
- Orquestração: todo ILegalAnalysisWorker registrado no DI é descoberto, executado em
  paralelo (MaxConcurrentWorkers 10) e incluído automaticamente no contexto do LLM. Contexto
  imutável; sem dependência entre workers; sem Buildings.Id. A ordem de execução e de
  registro NÃO define a ordem dos cards (a ordem pertence ao frontend).
- Limites atuais (não alterar): WorkerTimeout 60 s; RunTimeout 120 s compartilhado com o LLM;
  MaxPayloadBytes 1.000.000 por worker; MaxInputBytes do prompt 750.000; MaxConcurrentRuns 1.
- Coordenador: o orquestrador cria um LegalAnalysisSourceCoordinator por coleta e o entrega a
  cada worker em LegalAnalysisExecutionContext.SourceCoordinator (detalhes na seção 3.2).
- Persistência: Building -> Blob (manifest.json + report.json + workers.json) -> PropertyReport.
  NÃO criar arquivo novo no pacote nem migration/tabela/coluna.
- Prompt (LegalAnalysisReportPromptBuilder): envia ao LLM todos os workers com WorkerId,
  status, limitations e payload. Em 2026-11-06 só o Card 5 tem projeção compacta
  (FloodEnvironmentalRiskProjection.ForReport, ternário nas linhas 36–37); os demais workers
  entram com o payload como está. Se o total passar de MaxInputBytes, payloads inteiros são
  omitidos (maiores primeiro) e marcados como omitidos.
- API (LegalAnalysisWorkerPayloadMapper, linhas 12–14 em 2026-11-06): publica
  property-data-cache (bruto) e flood-environmental-risk (FloodEnvironmentalRiskProjection.
  ForApi). Qualquer outro worker recebe RawPayload null até ter projeção explícita.
- Falha do LLM: o backend devolve os workers com isComplete=false e NÃO persiste nada em SQL
  nem Blob (regra atual: exige relatório validado). Os cards continuam sendo montados a partir
  dos workers; o worker NUNCA depende do LLM.
- Frontend: LegalAnalysisResultDialogComponent compõe Property information (1) -> Recent
  Sales (3) + Active Listings (4) -> cards novos em ordem numérica (em 2026-11-06 só o
  Flood, 5) -> Summary (2, ainda SUMMARY_CARD_MOCK). Cada card novo entra depois do card de
  número anterior já implementado e antes do Summary.

---------------------------------------------------------------------------------------------
## 3. MECANISMOS DE REAPROVEITAMENTO (USO OBRIGATÓRIO) [REGRA FIXA]
---------------------------------------------------------------------------------------------
### 3.1 Cesta (cache de fontes existente)
- Leitura: IPropertyDataCacheReader.Read(detailId, listingId, address, zipCode) devolve as
  fontes em cache (mlsDetail, mlsSearchDetail, rentCastAvm, rentCastRent, regionalMarket,
  propertyComps). Somente leitura; não chama API. Ausência no cache NÃO prova ausência externa.
- Preenchimento: os clientes de integração (IOnlinePropertyDetail, IRentCastClient,
  IRealEstateApiClient) já fazem cache-aside com as regras atuais deles. Não altere chaves,
  formatos nem validade desse cache: o reader, os workers e a resolução do Building
  (LegalAnalysisReportPersistence) dependem deles.
- O cache serializa DTOs TIPADOS (camelCase); campos não declarados no DTO são descartados.
- Padrão: 1) ler a cesta; 2) se faltar e FetchMissingSources=true, buscar pela fonte (via
  coordenador quando compartilhada, seção 3.2); 3) reler a cesta; 4) usar o dado.

### 3.2 Coordenador de consultas por execução (Plano A)
- Arquivos: Application/Contracts/LegalAnalysisContracts.cs (ILegalAnalysisSourceCoordinator,
  propriedade SourceCoordinator, extensão QuerySourceAsync);
  Infrastructure/Orchestration/LegalAnalysisSourceCoordinator.cs;
  Infrastructure/Orchestration/LegalAnalysisSourceKeys.cs (em 2026-11-06 declara só
  MlsDetail e MlsListing); Infrastructure/Workers/SharedMlsQueries.cs.
- GARANTE: 1 consulta simultânea por chave dentro da mesma execução. Pedidos iguais enquanto
  ela roda aguardam a MESMA consulta; todos recebem o mesmo resultado ou o mesmo erro.
- NÃO GARANTE: nova consulta após falha, resultado ausente no cache ou cache expirado; nem
  deduplicação de chamadas de outras requisições (ex.: página do imóvel).
- Não guarda resultados: o controle da chave é liberado quando a consulta termina; o dado
  fica na cesta conforme as regras do cliente.
- Tokens: a consulta roda com o token da EXECUÇÃO; cada worker espera com o PRÓPRIO token.
  O timeout de um worker não cancela a consulta que outro ainda aguarda.
- Encerramento: no fim da coleta, consultas pendentes são canceladas e aguardadas por até 2 s;
  um pedido feito depois disso recebe OperationCanceledException.
- Uso no worker (uma linha): context.QuerySourceAsync(chave, token => cliente(..., token),
  tokenDeEspera). Sem coordenador no contexto (ex.: teste unitário), chama a fonte direto.
- MLS: NUNCA chame IOnlinePropertyDetail.GetAsync/GetByListingIdAsync diretamente em um
  worker. Use propertyDetail.DetailAsync(context, detailId, waitToken) e
  propertyDetail.ListingAsync(context, listingId, waitToken) (SharedMlsQueries). Eles montam
  a mesma chave para todos os workers. Resultado: Available | NotFound | Gone; falha de HTTP
  chega como exceção (HttpRequestException/JsonException).
- Critério de adoção para uma fonte nova:
  * fonte usada por 2 ou mais workers -> OBRIGATÓRIO passar pelo coordenador, com a chave
    declarada em LegalAnalysisSourceKeys (um método por fonte);
  * fonte usada por um único worker -> chama o cliente direto;
  * só entra no coordenador a fonte cujo contrato separa FALHA (exceção) de RESULTADO VÁLIDO
    (inclusive "sem resultados"). Se o contrato devolver null nos dois casos (ex.: AVM do
    RentCast), adapte o contrato antes.
- Chaves: "<rótulo>|<fonte>|<parâmetros>", com TODOS os parâmetros que mudam a resposta,
  exatos, sem espaços nas pontas e SEM arredondamento (inclusive coordenadas/geometria).
  O primeiro segmento (rótulo) é o único que vai para o log; os demais podem conter
  identificadores e nunca são gravados.
- Log automático (log/LegalAnalysis.txt + ILogger): Stage=SourceQuery,
  Status = Started | Joined | Completed | Failed | CancelledAtShutdown,
  detalhes source=<rótulo>;durationMs;failureType. Nunca identificador, endereço ou payload.

### 3.3 Identidade do imóvel (cada card resolve a SUA)
- Obrigatório (encaixe): ler a cesta com PropertyLookup + IPropertyDataCacheReader; MLS
  ausente buscado SOMENTE por SharedMlsQueries (3.2); reler a cesta. Assim todos os workers
  dividem a mesma consulta MLS da execução.
- De cada card: ler do registro MLS SOMENTE os campos de que o card precisa
  (PropertyLookup.FindObject/FindString). A falta de um campo que o card não usa nunca torna
  o card indisponível.
- Identidade é pré-requisito: entra no payload como estado de aplicabilidade/localização,
  nunca como dado exibido (endereço, unidade e tipo do imóvel pertencem ao Card 1).
- FloodPropertyLocationResolver é do Card 5: devolve null quando o imóvel não tem latitude e
  longitude válidas (linhas 55–57, conferido em 2026-11-06). Não o use em card que não precisa
  de coordenadas.
- Reaproveitamento inteligente: os cards novos que precisarem da sequência "ler cesta ->
  buscar MLS por SharedMlsQueries -> reler cesta" usam UM helper interno pequeno só com essa
  sequência (sem campos nem regras de card), criado pelo primeiro card novo que precisar.
  O Card 5 não é alterado pelo card novo; migrá-lo para o helper é refatoração separada,
  com autorização do arquiteto. Decisão registrada no PR.

### 3.4 Registro de estatutos (Florida Statutes)
- Todo card que exibe base legal usa UM registro interno, estático e versionado no código
  (sem banco, sem DI), com: seção (e subseção quando o card cita subseção), heading oficial,
  URL Online Sunshine, ano do estatuto ("2026 Florida Statutes"), regime a que se aplica
  (condominium = cap. 718, hoa = cap. 720, general) e o que a entrada rege.
- O primeiro card que precisar cria o registro; os seguintes só acrescentam entradas. Proibido
  um registro por card: a mesma lei não pode ter heading, URL ou ano diferentes em cards
  diferentes.
- A mesma seção só aparece em mais de um card com subseções diferentes, seguindo a citação do
  CSV de cada item. A mesma subseção é exibida por um único card; os outros a referenciam.
- Heading e URL só entram depois de conferidos no Online Sunshine; sem conferência a entrada
  não é publicada. Texto exibido: só heading oficial + link, sem interpretação jurídica.

### 3.5 Outros padrões já existentes
- Options: AddOptions<T>().Bind().Validate().ValidateOnStart() no
  NewLegalAnalysisModuleDependencyInjection (worker) e no IntegrationsDependencyInjection
  (cliente). TimeProvider para horário; nunca DateTime.UtcNow.
- Cliente HTTP de fonte nova: porta em NewLegalAnalysis.Application/Contracts (como as do
  Card 5); implementação em Integrations.Infrastructure/<Provedor>/Clients; registro com
  ConfigureEnvironmentalHttpClient (sem redirect, sem cookies, connect timeout 3 s, timeout
  controlado pelo worker). Integrations.Infrastructure JÁ referencia
  NewLegalAnalysis.Application: não criar ProjectReference novo.
- Log: ILogger + ILegalAnalysisRunTrace com RunId, sourceId e motivo; nunca endereço, nome de
  pessoa/associação, payload, chave, token ou URL com dados.

---------------------------------------------------------------------------------------------
## 4. CONTRATO DO WORKER (OBRIGATÓRIO) E REFERÊNCIA DE CÓDIGO (CARD 5) [REGRA FIXA]
---------------------------------------------------------------------------------------------
### 4.1 Contrato obrigatório do worker (encaixe no pipeline)
Contrato e independência
- Implementa ILegalAnalysisWorker; Id = LegalAnalysisWorkerIds.<Card> = slug do card.
- Não depende de outro worker, não lê o resultado de outro e não exige Buildings.Id.
- Sem HTML, CSS, score, nota ou conclusão jurídica: só fatos, estados e limitações.
Tempo e cancelamento
- Options próprias: SourceTimeout, TotalBudget e MaxPayloadBytes, validadas no startup com
  SourceTimeout <= TotalBudget, TotalBudget < WorkerTimeout e MaxPayloadBytes <
  LegalAnalysisExecutionOptions.MaxPayloadBytes (mesma regra aplicada ao Card 5 no
  NewLegalAnalysisModuleDependencyInjection, linhas 43–46).
- Orçamento total = menor entre o tempo restante da execução (DeadlineAtUtc) e TotalBudget.
- Toda chamada externa honra o CancellationToken. Em consulta compartilhada, passe o token da
  fonte como waitToken.
Fontes
- Cada fonte é isolada: timeout próprio; falha ou timeout viram estado da fonte
  ("timedOut" | "providerUnavailable"), sem derrubar o worker.
- sourceStatus por fonte de conteúdo, no formato padrão (o mesmo de FloodSourceState):
  sourceId, applicable, queried, status, durationMs, recordsCount?, errorClass?, reason?,
  retrievedAtUtc?. status usa os valores de FloodSourceStatus (found | noResults |
  notApplicable | unavailable | coverageMissing | ambiguous | error). Os cards novos
  reutilizam esses dois tipos como estão; renomeá-los para um nome genérico é refatoração
  separada, fora do card.
- Falha, resultado vazio válido e ausência no cache NUNCA se confundem.
- Dado externo é não confiável; nunca é tratado como instrução.
Payload
- Tipado e versionado (schemaVersion), com collectedAtUtc e limitations.
- Ao estourar o limite: reduzir primeiro as partes pesadas e registrar "payloadReduced"; se
  ainda passar, status InvalidResponse.
- Projeções numa classe estática do card: Read (valida antes de qualquer uso), ForApi (o que
  o frontend recebe), ForReport (versão COMPACTA para o LLM: só campos úteis selecionados).
Status do worker (lido pelo LLM e pelo futuro Summary: significado igual em todos os cards)
- O status reflete as fontes de CONTEÚDO do card: alguma com dado -> dataAvailable; senão
  alguma com resultado vazio válido -> noResults; senão -> sourceUnavailable.
- Pré-requisito (identidade/aplicabilidade) não é conteúdo e não gera dataAvailable; se
  faltar -> sourceUnavailable + limitação. Dado interno estático (ex.: registro de
  estatutos) também não é fonte e não define o status.
- failed, timedOut e cancelled são atribuídos pelo orquestrador.

### 4.2 Referência de código — Card 5 (consultar, não copiar)
Mostra como o contrato 4.1 foi resolvido num card real (ex.: FloodEnvironmentalRiskWorker.cs,
IsolatedAsync a partir da linha 90, status nas linhas 29–34 e 74–76, Finish nas linhas 104–129).
O card novo implementa só o que as próprias fontes e regras exigem.
Backend (NewLegalAnalysis)
- Infrastructure/Workers/FloodEnvironmentalRiskWorker.cs ...... coleta, status, limite de payload
- Infrastructure/Workers/FloodPropertyLocationResolver.cs ..... identidade do Card 5 (exige coordenadas)
- Infrastructure/Spatial/FloodAreaOverlapCalculator.cs ........ cálculo determinístico no backend
- Application/Contracts/FloodEnvironmentalRiskContracts.cs .... payload tipado + portas dos clientes
- Application/Contracts/FloodEnvironmentalRiskProjection.cs ... Read, ForApi, ForReport
- Application/Options/FloodEnvironmentalRiskOptions.cs ........ orçamento e limites próprios
- Api/NewLegalAnalysisModuleDependencyInjection.cs ............ validação das options
- Infrastructure/NewLegalAnalysisDependencyInjection.cs ....... registro explícito do worker
Backend (Integrations)
- Infrastructure/<Provedor>/Clients/*Client.cs ................ um cliente por fonte externa
- Api/IntegrationsDependencyInjection.cs ...................... options + ConfigureEnvironmentalHttpClient
Frontend (d_leevli-platform-web/src/app/features/listings)
- data-access/models/legal-analysis-flood-risk-payload.model.ts  (payload)
- data-access/models/legal-analysis-flood-risk.model.ts ........ (view model)
- data-access/models/legal-analysis-worker-payload.ts .......... (guard por workerId/schemaVersion)
- data-access/legal-analysis-flood-risk.mapper.ts + .spec + .fixture
- ui/legal-analysis/cards/flood-environmental-risk/ ............ (componente)
- pages/card-flood + rota "card-flood" em app.routes.ts ........ (laboratório, !production)
- legal-analysis-result-dialog (computed floodRiskCard + bloco @if) (composição)
Testes
- FloodEnvironmentalRiskWorkerTests, FloodEnvironmentalRiskContractsTests,
  FloodPropertyLocationResolverTests, FloodAreaOverlapCalculatorTests,
  LegalAnalysisSourceCoordinatorTests (coordenador), legal-analysis-flood-risk.mapper.spec.

### 4.3 O que NÃO reaproveitar do Card 5
- Endpoint de prévia por card: CollectFloodDataHandler (/collection/flood) executa TODOS os
  workers e tem nome inconsistente. Não criar endpoint novo; o laboratório usa fixture/mock.
- Cadeia de condicionais por workerId no PromptBuilder e no PayloadMapper: é o ponto de
  integração atual obrigatório (seção 5, itens 5 e 6), mas será substituída quando o Summary
  for implementado. Mantenha as projeções na classe estática do card para a migração ser
  mecânica.
- FloodPropertyLocationResolver em card que não precisa de coordenadas (3.3).
- Os clientes do Card 5 não têm cache; se a fonte passar a ser compartilhada, aplicar 3.2.

---------------------------------------------------------------------------------------------
## 5. CHECKLIST DE INTEGRAÇÃO DE UM CARD NOVO [REGRA FIXA]
---------------------------------------------------------------------------------------------
Antes de codar
[ ] 0. Ler AGENTS.md, MAPA_DE_ENTIDADES.txt, REGRAS_PRA_CRIAR_CODIGO.txt, LISTA_CARDS.txt,
       PROMPT_DEV_LEGAL.txt, Fontes_API_Legal_analyses.csv e os blueprints dos cards já
       especificados. Conferir duplicidade (⚠️ item 4).
Backend
[ ] 1. Constante em LegalAnalysisWorkerIds.
[ ] 2. Options do card + validação no NewLegalAnalysisModuleDependencyInjection (4.1).
[ ] 3. Registro explícito em AddNewLegalAnalysisWorkers (+ dependências).
[ ] 4. Fonte externa nova (se houver): cliente em Integrations + options + registro no
       IntegrationsDependencyInjection.
[ ] 5. Projeção ForReport no LegalAnalysisReportPromptBuilder. Em 2026-11-06 é um ternário só
       para o Card 5 (linhas 36–37): o primeiro card novo converte em switch por WorkerId,
       mantendo os casos existentes; os seguintes só acrescentam o seu caso.
[ ] 6. Projeção ForApi no LegalAnalysisWorkerPayloadMapper (mantendo os casos existentes).
[ ] 7. Fonte compartilhada com outro worker -> coordenador + chave em LegalAnalysisSourceKeys;
       MLS sempre por SharedMlsQueries; base legal sempre pelo registro de estatutos (3.4).
Frontend
[ ] 8. Modelo do payload + inclusão na união rawPayload (legal-analysis-result.model.ts,
       linha 57 em 2026-11-06).
[ ] 9. Guard por workerId/schemaVersion (legal-analysis-worker-payload.ts).
[ ] 10. Mapper + fixture + spec (data-access).
[ ] 11. Componente em ui/legal-analysis/cards/<card> (standalone, signals, colapsável:
        collapsed = signal(false), botão aria-expanded, paths 'm5 12 5-5 5 5'/'m5 7 5 5 5-5').
[ ] 12. Rota de laboratório pages/card-<x> fora do Shell, canMatch !environment.production,
        loadComponent.
[ ] 13. Composição no diálogo: computed do card + bloco @if após o card de número anterior já
        implementado e antes do Summary.
Testes
[ ] 14. Worker (status, isolamento, timeout, limite de payload, uso do coordenador), projeção
        (Read, ForApi, ForReport), cálculos, mapper e componente; suíte completa e
        WebApiBoundaryTests verdes; testes dos cards já implementados inalterados.

---------------------------------------------------------------------------------------------
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 7]
---------------------------------------------------------------------------------------------
Card único das REGRAS PRIVADAS e OBRIGAÇÕES da associação (HOA/COA) aplicáveis ao
proprietário e à unidade: locação, pets/ocupação/uso, reformas/elevador/áreas comuns e divisão
de manutenção. Todo dado exibido vem de fonte verificável e carrega documento, versão e data
de origem (PROMPT 7B). Variantes de layout mostram exatamente o mesmo view model.

Termos do card:
- Regra privada .... regra da associação (declaration, bylaws, rules, amendments,
                     architectural/board rules, maintenance matrix). Conteúdo do Card 7.
- Regra pública .... ordinance municipal/condado ou lei estadual de uso/locação. Não é
                     conteúdo do Card 7 (dono: Card 13).
- Tópico ........... grupo de regras do card: rental, petsOccupancyUse,
                     renovationCommonAreas, maintenanceResponsibility.

Fato determinante (CSV + código, conferido em 2026-11-06): as regras concretas de cada tópico
vivem SOMENTE em documentos privados da associação. O CSV marca os quatro itens com ressalva
— "Regras de aluguel" [VALIDAÇÃO LOCAL], "Regras de pets e ocupação" [ESCOPO PARCIAL],
"Restrições de reforma" [VALIDAÇÃO LOCAL], "Responsabilidade de manutenção" [VALIDAÇÃO LOCAL]
— e registra que os links do Clerk/Municode são diretórios, não documentos da associação.
Nenhum DTO MLS do codebase traz campo de regra (seção 7) e o NewLegalAnalysis não tem fluxo de
recebimento de documentos.

Fase 1 (este blueprint, 1 PR/1 card), conforme PROMPT_DEV_LEGAL.txt (7A–7F) e o CSV:
- Matriz de tópicos com estado honesto por tópico: "Association documents not reviewed" (sem
  documento, nenhuma regra é afirmada nem negada).
- Base legal por tópico -> Florida Statutes (registro único, 3.4): cap. 718 (condomínio) e
  cap. 720 (HOA) lado a lado, cada um rotulado, com a nota "Which one applies depends on the
  association's legal regime — see Card 6". O Card 7 não determina o regime.
- "Documents to request" por tópico -> lista fixa dos documentos que regem cada tópico,
  derivada da coluna "Fonte / serviço esperado" do CSV.
- Aplicabilidade -> estado da Flórida lido da cesta (seção 8). Fora da FL: estatutos FL ->
  notApplicable; a matriz continua com os mesmos estados.
- Ressalva de fair housing no tópico de pets/ocupação (CSV §760.23/§760.27: animal de
  assistência não é tratado como pet comum) — texto fixo, sem conclusão sobre o imóvel.
Fase 2 (backlog, card/PR próprio após homologação de cada fonte):
- Recebimento de documentos da associação (fluxo ÚNICO, compartilhado com Card 6 fase 2 e
  Card 15; não criar fluxo por card) + extração por tópico com evidência (documento, versão,
  data de vigência, página/cláusula), conflitos entre documentos/emendas preservados sem
  escolher vencedor (CSV "Conflitos entre documentos"), associação master x sub-association
  separadas (PROMPT_DEV_LEGAL, "ASSOCIAÇÕES").
- Declaration/amendments pelos Official Records do condado: o CSV só aponta o diretório do
  Miami-Dade Clerk; API/acesso programático não confirmado (gate 15.4).
- Uso de LLM para extração: em aberto (arquiteto Bruno, PROMPT 7E); nunca decide regra sem
  evidência documental.

Fronteira com outros cards (regra anti-duplicidade, ⚠️ item 4):
- Card 6 (HOA Financial Exposure) é o DONO de: existência/nome/regime da associação, taxa,
  special assessments, saúde financeira, SIRS, seguro master (§718.111(11)) e litígios.
  O Card 7 NÃO exibe nem determina nome, regime, taxa ou seguro e NÃO consulta o DBPR; mostra
  só o indicador "Association and legal regime: see Card 6". Responsabilidade por perda/seguro
  após sinistro também é do Card 6; o Card 7 trata apenas da divisão de manutenção
  ordinária/reparo.
- Card 13 (Permitted Use and Zoning) é o DONO das restrições PÚBLICAS de locação/STR (CSV
  "Restrições de locação pública": §509.032(7), Miami-Dade §33-28, ordinances). O Card 7
  exibe só o indicador "Public rental rules: see Card 13" (o CSV exige separar público de
  privado).
- Card 12 (Permits) é o DONO do permit público de obra (§553.79). O Card 7 exibe só o
  indicador "Public permit requirements: see Card 12" no tópico de reformas (CSV: aprovação
  da associação e permit público são exigências distintas).
- Card 14 (Rental and Investment) consome as regras de locação apenas por referência ao Card 7.
- Card 15 (Transaction Documents): disclosures HOA da transação (§720.401, §718.503) e
  conflitos entre documentos pós-oferta. Fora do Card 7.
- Card 1: confirmação da unidade condominial (§718.104). Card 9: Milestone/estrutura.
  Card 10: liens de associação registrados. Fora do Card 7.
- Conferido em 2026-11-06: nenhum card atual (1, 3, 4, 5) exibe regra de locação, pets,
  reforma ou manutenção; o Card 6 (ainda só blueprint) não as exibe por desenho. O Card 7 é o
  DONO desses dados no relatório.

Proibições invariantes: sem score, "Low Risk", "Rental friendly", "Pet friendly", semáforo ou
nota; sem placeholder em produção; ausência de documento NUNCA vira "no restrictions",
"rentals allowed" ou "pets allowed"; texto livre do anúncio (PublicRemarks) NUNCA vira regra
(seção 7); regra privada nunca é apresentada como regra pública (nem o inverso); responsável
por manutenção nunca presumido; nenhum limite de pets/ocupação aplicado sem ressalva de
acomodação razoável; estatuto exibido só com heading oficial + link, sem conclusão jurídica.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM EM 2026-11-06) [ESPECÍFICO DO CARD 7]
---------------------------------------------------------------------------------------------
- LegalAnalysisWorkerIds (Application/Contracts/LegalAnalysisContracts.cs:126) declara só
  PropertyDataCache e FloodEnvironmentalRisk. O Card 6 NÃO está implementado (nenhuma
  ocorrência de HoaFinancial/hoa-financial no backend nem no frontend); existe apenas como
  blueprint (raiz: BLUEPRINT_CARD_6_HOA_FINANCIAL_EXPOSURE.md V1; legal-share:
  CARD6_HOA_FIN_BLUEPRINT.md V3). Nenhuma ocorrência de hoa-rules/HoaRules no frontend.
- MLS — nenhum campo de regra:
  * RealEstateApiPropertyDto (mlsSearchDetail.listing.property) tem AssociationFee como único
    campo de associação; nenhum campo de pets, locação, prazo mínimo ou restrição.
  * mlsDetail.homeDetails (JsonElement preservado): as gravações reais em
    RealEstateApiOnlinePropertyDetailTests.cs (listings A12004619 e A12016406) contêm
    appliances, associationFeeFrequency, cooling, directions, exteriorFeatures, flooring,
    heating, lotFeatures, roof, sewer, taxAmount, watersource, zoning — nenhum campo de regra.
  * PublicRemarks (string) existe em RealEstateApiListingDto e RealEstateApiMlsDetailResponseDto.
    É texto comercial livre, não verificável e possivelmente desatualizado (CSV "Dados de
    anúncio e MLS": representação comercial que exige confirmação documental). NÃO é fonte
    do Card 7, nem por extração com LLM.
- Identidade: o estado do imóvel está em address.stateOrProvince do registro MLS (lido assim
  em FloodPropertyLocationResolver.cs:60). O resolvedor do Card 5 devolve null sem
  coordenadas (linhas 55–57) e por isso não é usado pelo Card 7 (seção 8).
- Status: LegalAnalysisWorkerStatus (LegalAnalysisContracts.cs:67) vai ao LLM em
  LegalAnalysisReportPromptBuilder.cs:79; o Card 5 calcula o status só pelas fontes de
  conteúdo (FloodEnvironmentalRiskWorker.cs:29–34 e 74–76).
- Recebimento de documentos: não existe no NewLegalAnalysis (as únicas ocorrências de
  "upload" são de BlobUpload do pacote de auditoria em RunLegalAnalysisHandler). Community
  tem ResidentDocument (prova de residência), que NÃO é documento da associação e não deve
  ser reaproveitado para isso.
- Estatutos: não há registro de estatutos no codebase (3.4 define o registro único).
- Frontend: rota card-flood em app.routes.ts (linhas 5–10) é o padrão de laboratório;
  rawPayload é união PropertyDataCacheWorkerPayload | FloodEnvironmentalRiskPayload
  (legal-analysis-result.model.ts:57); guard getFloodEnvironmentalRiskPayload
  (legal-analysis-worker-payload.ts:9); composição floodRiskCard
  (legal-analysis-result-dialog.ts:122, bloco @if em .html:64).
- Imagem de referência do layout do Card 7: AUSENTE em legal-share (existem só CARD5_* e
  CARD6_HOA_FIN_IMG_REF.jpg) -> gate 15.5.

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE E APLICABILIDADE [ESPECÍFICO DO CARD 7, SEGUINDO 3.3]
---------------------------------------------------------------------------------------------
- Resolução própria do Card 7, pelos mecanismos obrigatórios de 3.3: PropertyLookup +
  IPropertyDataCacheReader; com FetchMissingSources=true, MLS ausente buscado SOMENTE por
  SharedMlsQueries (DetailAsync/ListingAsync), compartilhando a consulta com os demais workers.
- Único campo lido: address.stateOrProvince (mlsDetail ou mlsSearchDetail.listing). O Card 7
  não precisa de coordenadas, condado nem tipo do imóvel; a falta deles nunca o torna
  indisponível. Nada disso é exibido (dono: Card 1).
- Não usar FloodPropertyLocationResolver (3.3).
- Aplicabilidade (pré-requisito, 4.1):
  * state = FL -> estatutos FL aplicáveis;
  * state != FL -> estatutos FL notApplicable;
  * state ausente -> unavailable + limitação "propertyStateUnavailable".
- Regime: o Card 7 não o determina (dono: Card 6). Exibe cap. 718 e 720 rotulados e a nota de
  regime (seção 6).
- propertyType NÃO decide se há associação (CSV: "Townhouse é tipologia, não regime legal";
  "cadastro DBPR não é lista universal de HOAs"). O card é exibido para todo imóvel, com a
  ressalva fixa "Applies only if the property is governed by an association".

---------------------------------------------------------------------------------------------
## 9. FONTES DA FASE 1 [ESPECÍFICO DO CARD 7]
---------------------------------------------------------------------------------------------
### 9.1 Cesta MLS — pré-requisito (estado), fonte COMPARTILHADA
- Leitura pela cesta, sem alterar nenhum DTO; busca quando ausente só via SharedMlsQueries.
- Nada da MLS é exibido no Card 7. Proibido chamar IOnlinePropertyDetail direto.
### 9.2 Documentos da associação — a fonte de CONTEÚDO do card
- Fase 1: indisponível (não existe fluxo de recebimento). sourceStatus:
  { sourceId:"association-documents", applicable:true, queried:false, status:"unavailable",
    reason:"noDocumentIntake" }.
- Fase 2: fluxo único de documentos (seção 6, gate 15.4).
### 9.3 Florida Statutes — entradas do Card 7 no registro único (3.4)
- Citações do CSV por tópico (HEADINGS E URLS A CONFERIR no Online Sunshine antes do código —
  gate 15.3):
  * rental ................... §718.110(13) (condominium); §720.306(1)(h) (hoa).
                               CSV: tratam de mudanças em restrições de locação; eficácia de
                               emendas varia com regime, data de aquisição e exceções legais.
  * petsOccupancyUse ......... §760.23; §760.27 (general — fair housing / acomodação).
                               CSV [ESCOPO PARCIAL]: não estabelecem limites gerais de pets ou
                               ocupação; sustentam só o enquadramento de não discriminação.
  * renovationCommonAreas .... §718.113(2) (condominium); §720.3035 (hoa).
                               §553.79 (permit público) NÃO entra: pertence ao Card 12.
  * maintenanceResponsibility  §718.113(1) (condominium); §720.303(1) (hoa).
                               §718.111(11) NÃO entra: é seguro, dono Card 6.
- §720.303: o Card 6 exibe as subseções (4)-(7) (financeiro, CSV "Saúde financeira"); o Card 7
  exibe só a subseção (1) (manutenção). Heading da seção já conferido pelo Card 6 em
  2026-11-02; conferir só a URL/âncora da subseção.
- Os estatutos são dado interno estático: não são fonte e não definem o status (4.1).
- "Documents to request" (lista fixa por tópico, do CSV "Fonte / serviço esperado"):
  * rental .................... declaration, bylaws, rules and amendments
  * petsOccupancyUse .......... declaration, bylaws, rules and amendments
  * renovationCommonAreas ..... declaration, architectural rules, board rules
  * maintenanceResponsibility . declaration, bylaws, maintenance matrix
  Exibidos como "Documents that usually govern this topic", sem afirmar que existem, que
  estão registrados ou que o comprador tem direito a recebê-los.
### 9.4 Florida DBPR — NÃO usado pelo Card 7
- O DBPR só serviria para descobrir o regime, que é dado do Card 6. Por isso o Card 7 não o
  consulta, em nenhuma fase.

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD (worker "hoa-rules-obligations", schemaVersion 1) [ESPECÍFICO]
---------------------------------------------------------------------------------------------
{
  schemaVersion, collectedAtUtc,
  applicability: { status, jurisdiction?:"FL" },
  topics[]: { topicId:"rental"|"petsOccupancyUse"|"renovationCommonAreas"|
                      "maintenanceResponsibility",
              status:"notReviewed",
              governingDocuments[]: códigos fixos (declaration, bylaws, rulesAndAmendments,
                                    architecturalRules, boardRules, maintenanceMatrix),
              legalReferences[]: { section, heading, url, statuteYear,
                                   regime:"condominium"|"hoa"|"general" },
              crossCardReferences[]: { card, reason } (ex.: {6,"associationAndRegime"},
                                     {13,"publicRentalRules"}, {12,"publicPermit"}),
              notices[]: códigos fixos (ex.: "reasonableAccommodation" em petsOccupancyUse,
                         "amendmentEffectVaries" em rental) },
  notCoveredByPublicSources[]: códigos fixos (rentalRules, petsOccupancyRules,
                               renovationApproval, maintenanceMatrix, ruleVersions,
                               amendments),
  sourceStatus[]: formato padrão (4.1),
  limitations[]
}
- applicability.status: valores de FloodSourceStatus (4.1); na fase 1 só found (FL),
  notApplicable (outro estado) ou unavailable (estado ausente).
- Status do worker (regra 4.1):
  * fase 1: sempre sourceUnavailable (a fonte de conteúdo — documentos — não existe) +
    limitação "associationDocumentsNotAvailable"; com estado ausente, também
    "propertyStateUnavailable";
  * fase 2: documentos com regras do tópico -> dataAvailable; documentos lidos sem o tópico ->
    noResults ("not found in the documents provided", nunca "no restriction").
- Status de tópico (enum próprio do card): fase 1 só "notReviewed"; a fase 2 acrescenta
  found | notFoundInDocuments | conflicting | ambiguous em schemaVersion 2, de forma aditiva,
  com rules[]:{ kind, valueRaw, appliesTo:"primary"|"secondary", evidence:{ documentType,
  version?, effectiveDate?, page?, clause? } }.
- Sem regime, nome ou registro da associação no payload (dono: Card 6).
- Sem payload bruto do provedor; payload alvo <= 30 KB (HoaRulesObligationsOptions.
  MaxPayloadBytes menor que o limite geral). Na fase 1 o payload é pequeno e determinístico.
- Projeções (classe estática HoaRulesObligationsProjection):
  * Read: valida schemaVersion e coleções antes de qualquer uso;
  * ForApi: payload público do card (por desenho, sem campos internos);
  * ForReport: resumo para o LLM — applicability, status por tópico, seções citadas,
    sourceStatus, limitations. Sem URLs, sem endereço, sem texto livre.

---------------------------------------------------------------------------------------------
## 11. BACKEND [ESPECÍFICO DO CARD 7, SEGUINDO 3, 4 E 5]
---------------------------------------------------------------------------------------------
- Application/Contracts/HoaRulesObligationsContracts.cs: payload tipado (sem porta de cliente
  na fase 1).
- Application/Contracts/HoaRulesObligationsProjection.cs: Read, ForApi, ForReport.
- Application/Options/HoaRulesObligationsOptions.cs: { SourceTimeout=8 s, TotalBudget=15 s,
  MaxPayloadBytes=30.000 }; validação no NewLegalAnalysisModuleDependencyInjection
  (SourceTimeout <= TotalBudget; TotalBudget < WorkerTimeout; MaxPayloadBytes < limite
  geral), conforme 4.1.
- Registro de estatutos: entradas do Card 7 no registro único (3.4); se ainda não existir,
  o Card 7 o cria.
- Infrastructure/Workers/HoaRulesObligationsWorker.cs:
  Id = LegalAnalysisWorkerIds.HoaRulesObligations ("hoa-rules-obligations"); registro
  explícito em AddNewLegalAnalysisWorkers. Identidade própria (seção 8), isolada com timeout,
  pela cesta + SharedMlsQueries; matriz de tópicos e estatutos montadas a partir do registro
  único. Status pela regra 4.1 (seção 10). TimeProvider; ILogger + ILegalAnalysisRunTrace sem
  endereço, nome de associação ou payload.
- LegalAnalysisReportPromptBuilder: switch por WorkerId (seção 5, item 5) com
  HoaRulesObligations -> HoaRulesObligationsProjection.ForReport; demais inalterados.
  O LLM nunca decide regra, regime ou aplicabilidade.
- LegalAnalysisWorkerPayloadMapper: acrescentar HoaRulesObligations ->
  HoaRulesObligationsProjection.ForApi, mantendo os casos existentes.
- Integrations: nenhuma alteração na fase 1.
- Não alterar: coordenador, orquestrador, cesta, clientes MLS/RentCast, persistência,
  contratos HTTP, outros workers.

---------------------------------------------------------------------------------------------
## 12. FRONTEND (Angular) [ESPECÍFICO DO CARD 7, SEGUINDO 4.2 E 5]
---------------------------------------------------------------------------------------------
- data-access/models/legal-analysis-hoa-rules-payload.model.ts (payload) e
  legal-analysis-hoa-rules.model.ts (view model).
- legal-analysis-result.model.ts: HoaRulesObligationsPayload na união de rawPayload.
- legal-analysis-worker-payload.ts: getHoaRulesObligationsPayload (workerId
  'hoa-rules-obligations' + schemaVersion 1), no padrão de getFloodEnvironmentalRiskPayload.
- data-access/legal-analysis-hoa-rules.mapper.ts + .spec + .fixture: payload -> view model
  (labels dos tópicos, estados, documentos, estatutos, indicadores de outros cards).
- ui/legal-analysis/cards/hoa-rules-obligations/: standalone, signals, colapsável (seção 5,
  item 11).
- Layout expandido (um bloco por dado, sem repetição; diagramação final pela imagem de
  referência — gate 15.5):
  1) Cabeçalho: título "HOA Rules and Obligations" + ressalva "Applies only if the property
     is governed by an association" + indicador "Association and legal regime: see Card 6".
  2) Painel "Rules Overview" (somente estados): 4 linhas, uma por tópico, com o estado
     "Association documents not reviewed". Sem semáforo, nota ou "allowed/not allowed".
  3) Um bloco por tópico (Rental · Pets, occupancy and use · Renovations, elevator and common
     areas · Maintenance responsibility): "Documents that usually govern this topic", "Florida
     law references" (cap. 718 e 720 rotulados + heading + link + nota de regime), notices
     fixos e indicadores de outros cards (Card 13 em Rental; Card 12 em Renovations).
  4) Rodapé: "Not covered by public sources" (lista fixa) + fontes/datas/limitações.
- Linha compacta: "HOA Rules and Obligations · Rental: not reviewed · Pets: not reviewed ·
  Renovations: not reviewed · Maintenance: not reviewed · Association documents required".
  Na fase 2, cada tópico mostra o estado documental (ex.: "found in declaration v<data>").
- Rota de teste: features/listings/pages/card-hoa-rules (path "card-hoa-rules", fora do Shell,
  canMatch !environment.production, loadComponent), no padrão de pages/card-flood. Variantes
  de layout só na rota de teste, todas consumindo o MESMO mock; após aprovação permanece
  apenas a variante aprovada.
- Composição no diálogo: computed hoaRulesCard + bloco @if após o bloco do card de número
  anterior já implementado (Card 6, ou Card 5 se o 6 ainda não existir) e antes do Summary.
- Regras ESLint de template do AGENTS.md; a11y teclado/leitor de tela; links oficiais com
  rel="noopener noreferrer". Sem PDF na fase 1.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE
---------------------------------------------------------------------------------------------
Backend (unit): FL / fora da FL / estado ausente; imóvel sem coordenadas (card montado
normalmente); status sempre sourceUnavailable na fase 1 com "associationDocumentsNotAvailable";
quatro tópicos sempre presentes com status notReviewed; nenhum valor de regra na fase 1;
cap. 718 e 720 sempre rotulados; nenhum campo de regime no payload; §553.79 e §718.111(11)
ausentes; §720.303 só com a subseção (1); indicadores de outros cards corretos; sem MLS;
orçamento/parcial com TimeProvider fake; isolamento da identidade (timeout/falha); limite de
payload; projeções Read/ForApi/ForReport; payload mapper; prompt sem dados sensíveis (sem URL,
endereço, nome).
Backend (reaproveitamento): MLS buscado por SharedMlsQueries (com e sem coordenador no
contexto, como em LegalAnalysisSourceCoordinatorTests); nenhuma chamada direta a
IOnlinePropertyDetail no worker; leitura pela cesta antes de buscar; nenhum uso de
FloodPropertyLocationResolver; registro de estatutos único (sem cópia por card).
Backend (integração): worker no orquestrador sem afetar property-data-cache, Flood (e Card 6);
WebApiBoundaryTests e suíte completa.
Frontend: specs do mapper, componente (expandido/compacto/fora da FL/sem estado) e guards;
nenhum texto "allowed"/"no restrictions" renderizado; smoke desktop/mobile com capturas no
work item.
Homologação real: condomínio em Miami-Dade; casa em HOA; imóvel fora da FL; MLS indisponível.
Conferir no log/LegalAnalysis.txt os eventos Worker e SourceQuery da execução (a consulta MLS
deve aparecer como Joined quando outro worker já a iniciou).
Aceite: cards já implementados e Summary inalterados; nenhum dado de outro card repetido;
nenhuma regra simulada ou inferida; todo item com fonte/data; nenhuma ausência tratada como
"sem regra" ou "permitido"; execução total < 120 s.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO
---------------------------------------------------------------------------------------------
1) Validação de dados e fontes: este documento + conferência anti-duplicidade (inclusive com
   o blueprint do Card 6) + gates 15.1, 15.3 e 15.5.
2) Rota card-hoa-rules com HTML estático nas versões expandida e compacta, usando só campos
   que a fase 1 entrega; aprovação visual.
3) Worker fase 1: options, registro no DI, registro de estatutos (3.4), projeções,
   PromptBuilder, PayloadMapper, modelo/guard/mapper/componente Angular, composição no
   diálogo.
4) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile, execução real e leitura do log.
5) Fase 2 (documentos) em card/PR próprio, após o gate 15.4.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
---------------------------------------------------------------------------------------------
15.1 Decidir o valor da fase 1: o card sai com aplicabilidade, base legal, documentos a
     solicitar e estados "not reviewed", sem nenhuma regra específica do imóvel (não existe
     fonte pública/integrada para isso). Alternativa: segurar o card até o fluxo de
     documentos (15.4). Recomendação: publicar a fase 1, porque cobre quatro itens do CSV com
     estado honesto e entrega um checklist acionável, sem inventar dados.
15.2 Peças compartilhadas com o Card 6 (registro de estatutos, switch do PromptBuilder, helper
     de MLS de 3.3 se houver): quem for implementado primeiro cria; o outro só acrescenta.
15.3 Conferir no Online Sunshine (2026 Florida Statutes) heading oficial e URL de §718.110,
     §720.306, §760.23, §760.27, §718.113 e §720.3035, e a URL/âncora de §720.303(1).
     Sem conferência, a entrada não é publicada.
15.4 Fase 2: definir o fluxo único de recebimento de documentos da associação
     (armazenamento, privacidade, retenção, quem envia) compartilhado com Card 6 fase 2 e
     Card 15; confirmar acesso programático ao Official Records (o CSV só aponta diretório);
     decidir o uso de LLM na extração (arquiteto Bruno).
15.5 Imagem de referência do layout do Card 7 (padrão legal-share/CARD7_HOA_RULES_IMG_REF.jpg)
     ainda não fornecida. Sem ela, o HTML estático segue a estrutura da seção 12, o STYLE
     GUIDE CARDS e os cards já implementados como referência visual.
15.6 Base de código (conferido em 2026-11-06 no repositório d_leevli-platform-api): o
     Plano A está commitado (57074dd6 "Modulariza Workers") na branch
     feat/legal-analysis-source-coordinator, publicada no remoto github e criada sobre a
     LEGAL local (1fd41f91), que contém o Card 5. Nem develop nem LEGAL contêm o
     coordenador; develop também não contém o Card 5; a LEGAL local difere de azure/LEGAL.
     A branch do card deve partir da branch de integração que contenha Card 5 + Plano A
     (+ Card 6, se já mergeado), depois do merge do Plano A.
15.7 Work item com critérios de aceite; branch feat/<card>-hoa-rules-obligations;
     commits feat(new-legal-analysis): #<card> ...

---------------------------------------------------------------------------------------------
## 16. ALINHAMENTO COM O SUMMARY (CARD 2) [REGRA FIXA]
---------------------------------------------------------------------------------------------
O Summary ainda não foi implementado (usa SUMMARY_CARD_MOCK) e não é alterado por este card.
Ele vai consumir de CADA worker, sem tratamento especial:
- Status, sourceStatus e limitations padronizados -> seções e nota calculadas por regras
  explícitas (PROMPT_DEV_LEGAL, item 2D), não pelo worker.
- Projeção ForReport -> contexto agregado enviado ao LLM.
- Id = slug do card -> mapeamento card -> worker (exceção: property-data-cache -> cards 1, 3, 4).
Por isso: o worker não calcula score, nota ou conclusão; entrega fatos e estados; e a falha do
LLM não afeta o worker nem o card.

---------------------------------------------------------------------------------------------
## 17. RISCOS CONHECIDOS FORA DO ESCOPO (NÃO CORRIGIR NO CARD) [REGRA FIXA]
---------------------------------------------------------------------------------------------
- O log do NewLegalAnalysis grava a exceção completa (exception.ToString()).
- Workers que ignoram cancelamento continuam rodando após o timeout (o orquestrador abandona
  a espera; o coordenador rejeita pedidos tardios).
- Trava do PropertyComps no RealEstateApiClient: corrida na remoção e nova consulta quando o
  resultado não vai ao cache.
- A persistência relê a identidade do cache para resolver o Building.
- LegalAnalysisWorkerCatalog é código morto; CollectFloodDataHandler executa todos os workers.
- O validador do relatório não confere se o relatório cobre todos os workers.
- MAPA_DE_ENTIDADES (seção 16) e PROMPT_DEV_LEGAL (estado atual) estão desatualizados quanto
  ao Card 5 e ao coordenador.
