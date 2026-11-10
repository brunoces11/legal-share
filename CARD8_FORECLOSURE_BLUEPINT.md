# LEEVLI | LEGAL ANALYSIS | CARD 8 — FORECLOSURE AND LEGAL PROCEEDINGS
# MASTER IMPLEMENTATION BLUEPRINT V1 — BASEADO NO TEMPLATE DO CARD 6 (CARD6_HOA_FINANCIAL.md, V3)
# (aderente ao código em 2026-11-10, incluindo o Plano A: coordenador de consultas por execução)

Este documento é o blueprint de implementação do Card 8 (Foreclosure and Legal Proceedings).
Segue a estrutura do template do Card 6: as seções [REGRA FIXA] são idênticas às do template;
as seções [ESPECÍFICO DO CARD] são próprias do Card 8.

V1 (2026-11-10): a IMAGEM DE REFERÊNCIA do card ("Layout 2") é o contrato visual. Todo campo
mostrado na imagem está mapeado, na seção 6.1, para um campo real de fonte programática cuja
documentação foi lida em 2026-11-10 (RealEstateAPI Property Detail e APIs do Miami-Dade Clerk).
As seções [REGRA FIXA] foram reconferidas contra o código em 2026-11-10: as referências de
arquivo e linha datadas de 2026-11-06 continuam válidas.

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
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 8]
---------------------------------------------------------------------------------------------
Card único dos PROCESSOS JUDICIAIS e dos SINAIS DOCUMENTADOS DE FORECLOSURE ligados ao imóvel:
o processo (número, tribunal, tipo, partes, status), o registro que o torna público (lis
pendens, data de registro, CFN/book-page, descrição legal, processo relacionado) e os
indicadores do provedor (pre-foreclosure, auction, REO, tipo e data do documento). Todo dado
vem de fonte programática e carrega a origem; nada é inferido.

Termos do card:
- Processo ......... caso civil publicado pelo Clerk do condado (case).
- Registro ......... instrumento gravado nos Official Records do condado (ex.: lis pendens).
- Indicador ........ sinal booleano de provedor agregado (RealEstateAPI). Não é prova: o CSV
                     diz "Indicador de provedor não comprova execução vigente ou perda da
                     propriedade; confirmar o estágio nos autos".
- Documento ........ aviso de foreclosure mais recente publicado pelo provedor
                     (foreclosureInfo/auctionInfo).
- Folio ............ número do parcel no condado (Miami-Dade: 13 dígitos). Chave da consulta
                     ao Clerk; não é exibido (dono: Card 1).

### 6.1 CONTRATO VISUAL — IMAGEM DE REFERÊNCIA -> DADOS
Imagem recebida em 2026-11-10: traz SOMENTE a versão expandida, variante "Layout 2". O chip
"Illustrative data" e o rótulo "Layout 2" são marcas do laboratório, não dados (seção 12).
A versão compacta não está na imagem e é definida na seção 12 (Regra 4 do PROMPT_DEV_LEGAL).
A imagem tem 3 blocos + rodapé. Cada linha abaixo é um campo da imagem:
"campo (exemplo da imagem) <- fonte primária | alternativa | estado quando ausente".

Bloco A — "Case overview" (processo)
- Case number (2024-CA-012345)
    <- Clerk Official Records: CASE_NUM do registro de lis pendens, confirmado por
       Clerk Civil: caseNumber
    |  RealEstateAPI: auctionInfo.caseNumber (e foreclosureInfo[].caseNumber, gate 15.1)
    |  "Not available"
- Court (Example County Circuit Court)
    <- Clerk Civil: courtLocation + juditialSection (grafia da API) + tipo do tribunal lido do
       próprio número do processo (CA = Circuit Civil, CC = County Civil; tabela fixa, 15.2)
    |  sem alternativa (o provedor não publica tribunal)  |  "Not available"
- Case type (Foreclosure)
    <- Clerk Civil: caseType (+ caseTypeCode)  |  sem alternativa  |  "Not available"
- Parties (ABC Lending, LLC v. Jackson Rivera)
    <- Clerk Civil: caseStyle (texto publicado pelo Clerk, sem reescrita)
    |  Clerk Official Records: FIRST_PARTY / SECOND_PARTY do registro de lis pendens
    |  RealEstateAPI: lenderName, exibido só como "<lender> (lender reported)"; nunca
       completado com o nome do proprietário  |  "Not available"
- Case status (pill "Open")
    <- Clerk Civil: caseStatus (+ dispositionCode / dispositionDate)
    |  sem alternativa (foreclosureInfo.active NÃO é status de processo)  |  "Not available"

Bloco B — "Filing, recording & property details" (registro)
- Filing date (Feb 14, 2024)
    <- Clerk Civil: filingDate  |  sem alternativa  |  "Not available"
- Lis pendens (pill "Yes")
    <- Clerk Official Records: existe registro de lis pendens para o folio (DOC_TYPE) sem
       liberação vinculada (ORIG_CFN_YEAR/ORIG_CFN_SEQ ou ORIG_REC_BOOK/ORIG_REC_PAGE)
    |  RealEstateAPI: foreclosureInfo[] com aviso de lis pendens (noticeType NOL, 15.1)
    |  "Not found" (consulta feita, nada encontrado) ou "Unavailable" (fonte indisponível)
- Recording date (Feb 20, 2024)
    <- Clerk Official Records: REC_DATE  |  RealEstateAPI: foreclosureInfo[].recordingDate
    |  "Not available"
- CFN / book-page (CFN 2024R-123456 / BK 34000 PG 5678)
    <- Clerk Official Records: CFN_YEAR + "R" + CFN_SEQ / REC_BOOK + REC_PAGE (+ BOOK_TYPE)
    |  sem alternativa (o provedor não publica CFN nem book/page)  |  "Not available"
- Legal description (Lot 12, Block 4, Example Estates, ... Plat Book 10, Page 20)
    <- Clerk Official Records: LEGAL_DESCRIPTION do registro de lis pendens (+ SUBDIV_NAME,
       BLOCK_NO, PLAT_BOOK, PLAT_PAGE como referência de plat)
    |  RealEstateAPI: lotInfo.legalDescription (rotulada "per county assessor data")
    |  "Not available"
- Related case (2023-CA-098765)
    <- Clerk Civil: consolidatedCase
    |  Clerk Official Records: outro CASE_NUM em registros de lis pendens do mesmo folio
    |  linha omitida quando não houver (não é "Not available": ausência é o estado normal)

Bloco C — "Case indicators & document" (provedor)
- Pre-foreclosure (pill "Not confirmed")  <- RealEstateAPI PropertyDetail: preForeclosure
- Auction (pill "Not confirmed")          <- RealEstateAPI PropertyDetail: auction
- REO (pill "Not confirmed")              <- RealEstateAPI: reo (documentado na Property
                                             Search; presença na PropertyDetail -> gate 15.1)
    Estados dos três: true -> "Reported"; false -> "Not confirmed"; chave ausente ou fonte
    indisponível -> "Unavailable". Nunca "Confirmed", "Clear" ou "No".
- Document type (Notice of Lis Pendens)
    <- RealEstateAPI: documentType do aviso mais recente (foreclosureInfo[] / auctionInfo)
    |  Clerk Official Records: DOC_TYPE do registro principal  |  "Not available"
- Document date (Feb 20, 2024)
    <- RealEstateAPI: recordingDate do MESMO aviso (dateKind "recording")
    |  Clerk Official Records: DOC_DATE do registro principal (dateKind "document")
    |  "Not available"
    Tipo e data saem sempre em par, da mesma fonte e do mesmo instrumento.

Rodapé — "Source: relevant county records / RealEstateAPI"
    <- montado a partir de sourceStatus: nomes das fontes que responderam + data da consulta.
       Texto fixo é proibido: se o Clerk não respondeu, o rodapé não cita "county records".

Regra de leitura do contrato: fonte primária e alternativa NUNCA são fundidas no mesmo campo;
cada campo sai com um único sourceId. Nos blocos A e B, se as duas existirem e divergirem,
vale a do Clerk e o payload registra a limitação "sourcesDiverge".

### 6.2 Fases
Fase 1 (este blueprint, 1 PR/1 card), conforme PROMPT_DEV_LEGAL.txt (8A–8F) e o CSV
("Foreclosure e ações sobre o imóvel" e "Lis pendens"):
- RealEstateAPI Property Detail (qualquer condado dos EUA): indicadores, documento, data de
  registro, número do processo quando publicado, descrição legal alternativa.
- Miami-Dade Clerk — Official Records por folio e Civil por número do processo (somente
  imóveis em Miami-Dade): todos os campos dos blocos A e B.
- Fora de Miami-Dade o card é montado com os campos do provedor e os demais ficam "Not
  available", com a limitação "countyClerkNotIntegrated". O CSV só mapeia o Clerk de Miami-Dade.
Fase 2 (backlog, card/PR próprio após homologação de cada fonte): Clerks de outros condados
(Broward primeiro); linha do tempo do processo (api/Civil dockets); imagens de documentos;
arquivos bulk por FTP; busca por nome de parte (não existe na API atual); julgamentos e liens
involuntários (RealEstateAPI Reports/PropertyLiens, add-on pago — dono: Card 10).

### 6.3 Fronteira com outros cards (regra anti-duplicidade, ⚠️ item 4)
- Card 1 (Property Identity) já EXIBE proprietário e parcel/APN
  (legal-analysis-property-card.mapper.ts:85–89). O Card 8 usa o folio só como chave de
  consulta e não exibe folio, endereço nem proprietário. "Parties" é o caseStyle do processo
  (dado do tribunal), nunca derivado do campo de proprietário.
- Card 6 (HOA Financial Exposure) é o DONO de "Litígios da associação" (fase 2 do Card 6,
  mesma fonte api/Civil). O Card 8 não lista processos da associação. Um processo encontrado
  NO FOLIO do imóvel em que a associação é parte (ex.: cobrança §718.116) é processo do imóvel
  e fica no Card 8, exibido como o Clerk publica, sem dado financeiro da associação.
- Card 10 (Ownership and Public Records) é o DONO de escritura, hipotecas, liens, julgamentos
  e servidões. O PROMPT cita "lis pendens" nos cards 8 (8B/8C) e 10 (10B): fica decidido aqui
  que o Card 8 é o DONO do lis pendens (existência, data, CFN/book-page e descrição legal DO
  instrumento de lis pendens); o Card 10 só o referencia ("Lis pendens: see Card 8").
  As flags `judgment` e `lien` da PropertyDetail pertencem ao Card 10 e não entram aqui.
- Card 11 (Taxes): tax lien, tax certificate e tax deed. Fora do Card 8.
- Card 15 (Transaction Documents): title commitment e exceções. Fora do Card 8.
- Conferido em 2026-11-10: nenhum card atual (1, 3, 4, 5) exibe processo, lis pendens, CFN,
  descrição legal ou indicador de foreclosure. O Card 8 é o DONO desses dados no relatório.
- Página do imóvel: LeadTypeCuration (#357) remove foreclosure/preForeclosure/auction da API
  pública do anúncio por decisão de produto. O Card 8 NÃO altera essa curadoria: os
  indicadores aparecem só no relatório Legal Analysis (execução autenticada) — decisão 15.4.

Proibições invariantes: sem score, semáforo, "Low risk", "Clean", "No foreclosure" ou "Clear
title"; sem placeholder em produção; null nunca vira false; ausência de registro NUNCA vira
"sem processo" (CSV: "Busca sem resultado não assegura ausência"); indicador do provedor
nunca é apresentado como confirmação; alegação nunca é apresentada como decisão; nenhuma
conclusão sobre validade, expiração ou efeito do lis pendens (§48.23); partes nunca montadas
com dados de fontes diferentes; processo nunca escolhido arbitrariamente quando houver mais
de um; valores financeiros do aviso (defaultAmount, judgmentAmount, openingBid,
estimatedBankValue, originalLoanAmount) não entram no payload — a imagem não os exibe; nome de
pessoa nunca vai para log nem para o LLM.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM EM 2026-11-10) [ESPECÍFICO DO CARD 8]
---------------------------------------------------------------------------------------------
- LegalAnalysisWorkerIds (Application/Contracts/LegalAnalysisContracts.cs:126–130) declara só
  PropertyDataCache e FloodEnvironmentalRisk. Os Cards 6 e 7 NÃO estão implementados (só
  blueprints em legal-share). O Card 8 pode ser o primeiro card novo: nesse caso cria o switch
  do PromptBuilder (seção 5, item 5) e o helper de MLS de 3.3.
- RealEstateAPI — endpoints integrados hoje: MLSSearch, MLSDetail e PropertyComps
  (RealEstateApiClient), PropertyParcel /v1 (RealEstateApiParcelClient) e AutoComplete.
  /v2/PropertyDetail NÃO está integrado. IRealEstateApiClient só tem GetPropertyCompsAsync.
- Cesta (mlsSearchDetail = RealEstateApiMlsListingItemDto): traz `id` (id do imóvel no
  provedor; lido pelo Card 5 em FloodPropertyLocationResolver.ResolveProviderPropertyId,
  linhas 195–204), `public.apn`, `public.propertyId` e `public.leadTypes`
  (Dictionary<string, object>). Nenhum DTO MLS tem número de processo, tribunal, documento,
  CFN ou descrição legal.
- `public.leadTypes`: as chaves foreclosure, preForeclosure e auction só aparecem em fixtures
  de teste (RealEstateApiMlsSearchItemMapperTests.cs:366–381;
  RealEstateApiOnlinePropertyDetailTests.cs:1029); não há `reo`. O formato real não está
  gravado. O Card 8 NÃO lê leadTypes: os indicadores vêm da PropertyDetail (fonte única).
- Divergências FORA do escopo (não corrigir neste PR, reportar ao arquiteto):
  RealEstateApiMapper.ExtractBoolFlag não trata JsonElement, então PublicLeadTypes sai null
  com payload real (comentário em RealEstateApiOnlinePropertyDetailTests.cs:319–323);
  PropertyDetailDto.PreForeclosure (linha 50) e property-detail.mapper.ts:277 leem
  preForeclosure para a página do imóvel.
- LeadTypeCuration (Properties.Application/Listings/LeadTypeCuration.cs:47–52) zera
  Foreclosure, PreForeclosure e Auction na API do anúncio. Não é alterado.
- Miami-Dade Clerk: nenhum cliente no codebase. A pasta Integrations.Infrastructure/MiamiDade
  já existe (MiamiDadeEvacuationZoneClient, Card 5) e normaliza o condado nas linhas 16–20
  ("MIAMI-DADE", "MIAMI-DADE COUNTY", "MIAMI DADE", "MIAMI DADE COUNTY").
- Estado e condado do imóvel: address.stateOrProvince e address.countyOrParish do registro
  MLS (lidos assim em FloodPropertyLocationResolver.cs:60).
- ConfigureEnvironmentalHttpClient (IntegrationsDependencyInjection.cs:419–426) já aplica
  RemoveAllLoggers: a URL da requisição não é registrada pelos loggers do HttpClient
  (requisito para o AuthKey do Clerk, que vai na query string — seção 9.3).
- Registro de estatutos: não existe no codebase. O Card 8 não precisa dele (9.5).
- Frontend: rota de laboratório card-flood (app.routes.ts:5–10); rawPayload é a união
  PropertyDataCacheWorkerPayload | FloodEnvironmentalRiskPayload
  (legal-analysis-result.model.ts:57); guard getFloodEnvironmentalRiskPayload
  (legal-analysis-worker-payload.ts:9–12); composição floodRiskCard em
  pages/listing-detail/components/legal-analysis-result-dialog (.ts:122; bloco @if em
  .html:64; Summary em .html:71 com sampleData). Não existe chip "Illustrative data"
  reutilizável; o Summary tem o rótulo "Sample data" (summary-card-preview.html:30–31).
- SUMMARY_CARD_MOCK já tem a seção 'Foreclosure & Legal Process' (mock; não é alterada).
- Imagem de referência do Card 8: AUSENTE em legal-share (recebida na conversa) -> gate 15.6.
- log/LegalAnalysis.txt: execução mais recente 200c0573 (2026-11-06) concluída com os dois
  workers atuais; é a linha de base para comparar depois do Card 8.

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE E APLICABILIDADE [ESPECÍFICO DO CARD 8, SEGUINDO 3.3]
---------------------------------------------------------------------------------------------
- Resolução própria do Card 8, pelos mecanismos obrigatórios de 3.3: PropertyLookup +
  IPropertyDataCacheReader; com FetchMissingSources=true, MLS ausente buscado SOMENTE por
  SharedMlsQueries (DetailAsync/ListingAsync), compartilhando a consulta com os demais workers.
- O registro MLS é só pré-requisito no Card 8: nada dele é exibido.
- Campos lidos:
  * address.stateOrProvince e address.countyOrParish -> jurisdição do Clerk;
  * `id` do item de mlsSearchDetail -> id do imóvel no provedor (chave da PropertyDetail),
    aceito só quando o listingId do item é o esperado — mesma regra do Card 5 (linhas
    195–204). O método do Card 5 é privado e o resolvedor exige coordenadas, então o Card 8
    faz a própria leitura; unificar é refatoração separada (registrar no PR);
  * public.apn -> candidato a folio.
- Coordenadas, endereço, tipo do imóvel e proprietário NÃO são necessários: a falta deles
  nunca torna o card indisponível. Não usar FloodPropertyLocationResolver (3.3).
- Folio: somente dígitos de public.apn; a PropertyDetail também publica lotInfo.apn e
  lotInfo.apnUnformatted. Miami-Dade exige 13 dígitos. Regras: uma fonte só -> usa; duas
  iguais -> usa; duas diferentes -> não consulta por folio (ambiguous, "folioMismatch");
  nenhuma ou formato inválido -> não consulta por folio ("folioUnavailable").
- Aplicabilidade (pré-requisito, 4.1):
  * id do provedor ausente -> PropertyDetail não consultada (unavailable,
    "providerPropertyIdUnavailable");
  * FL + Miami-Dade -> Clerk aplicável;
  * FL + outro condado -> Clerk coverageMissing ("countyClerkNotIntegrated");
  * condado ausente -> Clerk coverageMissing ("countyUnconfirmed");
  * fora da FL -> Clerk notApplicable; a PropertyDetail continua (cobertura nacional).

---------------------------------------------------------------------------------------------
## 9. FONTES DA FASE 1 [ESPECÍFICO DO CARD 8]
---------------------------------------------------------------------------------------------
### 9.1 Cesta MLS — pré-requisito, fonte COMPARTILHADA
- Leitura pela cesta, sem alterar nenhum DTO; busca quando ausente só via SharedMlsQueries
  (3.2). Proibido chamar IOnlinePropertyDetail direto.
### 9.2 RealEstateAPI — Property Detail (conteúdo; todos os condados)
- POST https://api.realestateapi.com/v2/PropertyDetail, body { "id": <id do provedor> },
  header x-api-key (RealEstateApiOptions já existente). Documentação lida em 2026-11-10:
  developer.realestateapi.com/reference/property-detail-api-1 e
  /reference/property-detail-response-object.
- Campos DOCUMENTADOS usados pelo card (data.*):
  * raiz: id, preForeclosure, auction, lastUpdateDate;
  * auctionInfo (só vem com auction=true): active, auctionDate, caseNumber, documentType,
    noticeType, recordingDate;
  * foreclosureInfo[] (inclui documentos históricos, não só o ativo): foreclosureId,
    recordingDate, documentType, typeName, active;
  * lotInfo: legalDescription, apn, apnUnformatted;
  * propertyInfo.address: county, fips.
- Documentado em outra página, a confirmar na PropertyDetail (gate 15.1): `reo` e
  `foreclosure` (resposta da Property Search); enum notice_type FOR | NOD | NOL | NTS | REO
  (filtro da Property Search sobre a data de registro de .foreclosureInfo).
- lenderName é lido só como alternativa de "Parties" (6.1). Valores financeiros, telefone,
  trustee, dados de proprietário, hipoteca e equity NÃO são lidos.
- Leitura tolerante por JsonElement (o provedor mistura string e número e dois formatos de
  data: "2022-10-05" e ISO com hora). Não criar classe tipada da resposta inteira — o codebase
  já registrou esse problema no MLSDetail.
- Documento principal: avisos com active=true; entre eles o de recordingDate mais recente.
  Sem ativo -> o mais recente de todos, marcado historical=true (a própria documentação
  manda conferir .recordingDate porque há foreclosures históricas inativas).
- Estado da fonte: found = algum indicador true ou algum aviso; noResults = imóvel
  encontrado sem indicador e sem aviso (vazio válido); 404 -> noResults "propertyNotFound";
  402/429/5xx/timeout -> unavailable.
- Contrato do cliente: separa FALHA (exceção) de RESULTADO VÁLIDO (found | notFound). Hoje só
  o Card 8 usa a fonte -> chamada direta (3.2). Os cards 10 e 11 têm itens do CSV que a mesma
  resposta cobre; quando um deles passar a usá-la, a fonte vira compartilhada: chave
  LegalAnalysisSourceKeys.PropertyDetail(externalSource, propertyId) e coordenador.
- Sem cache e sem nova tentativa na fase 1: 1 chamada (1 crédito) por execução.
### 9.3 Miami-Dade Clerk — Commercial Data Services (conteúdo; só Miami-Dade)
- Portal: https://www2.miamidadeclerk.gov/Developers/ ; documentação /Developers/Help, lida
  em 2026-11-10. Chamadas: GET sobre www2.miamidadeclerk.gov/Developers/, com AuthKey.
- (a) Official Records por folio:
  GET api/OfficialRecords?parameter1={folio}&parameter2=FN&authKey={key}
  (a mesma rota aceita CFN: parameter1=ano, parameter2="R"+sequência; e book/page).
  Campos do registro (amostra XML da documentação): CFN_YEAR, CFN_SEQ, DOC_TYPE, REC_DATE,
  DOC_DATE, REC_BOOK, REC_PAGE, BOOK_TYPE, FIRST_PARTY, SECOND_PARTY, PARTY_CODE,
  PARTY_SEQUENCE, ORIG_CFN_YEAR, ORIG_CFN_SEQ, ORIG_REC_BOOK, ORIG_REC_PAGE, LINK_DOCTYPE,
  BLOCK_NO, PLAT_BOOK, PLAT_PAGE, SUBDIV_NAME, LEGAL_DESCRIPTION, FOLIO_NUMBER, CASE_NUM,
  STATUS.
- (b) Processo civil por número:
  GET api/Civil?caseNumber={caseNumber}&AuthKey={key}
  Campos de CaseInfo: caseNumber, stateCaseNumber, filingDate, caseStyle, caseTypeCode,
  caseType, juditialSection, courtLocation, consolidatedCase, dispositionCode,
  dispositionDate, caseStatus.
- Envelope das duas: Status, StatusDesc, UnitsBalance, IPAddress.
- NÃO usados na fase 1: dockets (api/Civil?civilCaseNumber=), imagens (docketNumber), FTP.
- Custo: US$ 0,20 por unidade, 1 unidade por requisição, saldo pré-pago. Teto por execução:
  2 requisições (1 Official Records + 1 Civil) = US$ 0,40, imposto por
  MaxClerkRequestsPerRun.
- Acesso: conta de Developer habilitada + Access Request Form notarizado. O portal anuncia
  nova versão da aplicação Developers para 2026-12-01, com documentação atualizada "em
  breve" -> gate 15.2 refaz a conferência depois dessa data.
- Segredo: o AuthKey vai na query string por desenho da API. Fica em options (user-secrets /
  cofre), nunca no repositório, em log, em mensagem de exceção ou no payload. A exceção do
  cliente não carrega a URL. Conferir no teste que a telemetria de dependências não grava a
  query (QueryStringRedactionTelemetryProcessor).
- Algoritmo determinístico:
  1. Official Records por folio -> manter linhas cujo DOC_TYPE esteja na tabela de tipos de
     lis pendens e de liberação (códigos reais no gate 15.2).
  2. A API devolve uma linha por parte (PARTY_SEQUENCE): agrupar por instrumento
     (CFN_YEAR + CFN_SEQ).
  3. Lis pendens liberado = existe liberação cujo ORIG_CFN_* ou ORIG_REC_BOOK/PAGE aponta
     para ele.
  4. Registro principal = lis pendens NÃO liberado, com CASE_NUM, de REC_DATE mais recente;
     se todos estiverem liberados, o mais recente (lisPendens = "released").
  5. Civil com o CASE_NUM do registro principal. Sem registro por folio e com número de
     processo publicado pelo provedor -> Civil com esse número (o card continua sem CFN).
  6. relatedCases = consolidatedCase do processo principal + outros CASE_NUM dos registros
     de lis pendens do folio; otherCasesCount informa o total.
- Estado das fontes: Official Records com 0 lis pendens -> noResults ("No lis pendens found
  in the records searched"; não significa "sem processo"); Civil sem o processo -> noResults;
  chave ausente/desabilitada -> unavailable "notConfigured"; saldo zerado -> unavailable
  "insufficientUnits"; erro/timeout -> unavailable.
### 9.4 Clerks de outros condados — NÃO usados na fase 1
- O CSV só mapeia Miami-Dade. Cada condado novo entra na fase 2 como cliente próprio atrás da
  mesma porta, depois de homologado (acesso, custo, campos).
### 9.5 Florida Statutes — NÃO exibidos pelo Card 8
- O CSV cita §702.01 e §48.23 como base legal dos dois itens. A imagem de referência não tem
  bloco de base legal: o card não exibe estatutos e não cria entradas no registro único
  (3.4) na fase 1. Se o produto pedir, entram pelo registro único, com heading/URL conferidos.

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD (worker "foreclosure-legal-proceedings", schemaVersion 1) [ESPECÍFICO]
---------------------------------------------------------------------------------------------
{
  schemaVersion, collectedAtUtc,
  jurisdiction: { status, state?, clerk?:"miami-dade",
                  clerkCoverage:"integrated"|"notIntegrated"|"unconfirmed"|"notApplicable" },
  case: { status, sourceId?,
          caseNumber?, stateCaseNumber?,
          court?:{ divisionCode?, divisionName?, sectionRaw?, locationRaw? },
          caseTypeRaw?, caseTypeCode?,
          parties?:{ text, kind:"caseStyle"|"recordParties"|"lenderOnly", sourceId },
          caseStatus?:"open"|"closed"|"reopened"|"other", caseStatusRaw?,
          dispositionCode?, dispositionDate?, filingDate?,
          relatedCases[]:{ caseNumber, relation:"consolidated"|"otherCaseOnParcel" },
          otherCasesCount },
  recording: { status, sourceId?,
               lisPendens:"recorded"|"released"|"notFound"|"unavailable",
               recordingDate?, cfn?:{ year, sequence }, book?, page?, bookType?,
               legalDescription?:{ text, sourceId },
               platReference?:{ subdivision?, block?, platBook?, platPage? } },
  indicators: { status, sourceId?, asOf?,
                preForeclosure, auction, reo: "reported"|"notReported"|"unavailable" },
  document: { status, sourceId?, typeRaw?, noticeTypeCode?, date?,
              dateKind?:"document"|"recording", historical },
  sourceStatus[]: formato padrão (4.1),
  limitations[]
}
- sourceId: "realestateapi-property-detail" | "miami-dade-clerk-official-records" |
  "miami-dade-clerk-civil". status das seções: valores de FloodSourceStatus (4.1).
- Datas do Clerk e do provedor são datas de calendário: gravar como data (sem hora e sem
  conversão de fuso). asOf = lastUpdateDate do provedor; collectedAtUtc é só o horário da
  coleta.
- Status do worker (regra 4.1): as três fontes acima são fontes de conteúdo. Alguma found ->
  dataAvailable; senão alguma noResults -> noResults; senão -> sourceUnavailable. A cesta MLS
  e a jurisdição são pré-requisito e não geram dataAvailable. Em noResults o card é montado
  com "Not confirmed" / "Not found" e a limitação "absenceIsNotProof".
- Limitações (códigos fixos): providerIndicatorNotProof (sempre que houver indicador),
  absenceIsNotProof, countyClerkNotIntegrated, countyUnconfirmed, folioUnavailable,
  folioMismatch, providerPropertyIdUnavailable, sourcesDiverge, historicalDocumentOnly,
  multipleCasesOnParcel, payloadReduced.
- Sem payload bruto das fontes, sem folio, sem endereço e sem proprietário. Payload alvo
  <= 20 KB (MaxPayloadBytes 30.000). Partes pesadas, reduzidas primeiro: legalDescription.text
  (corte em 1.000 caracteres) e parties.text (corte em 300), com "payloadReduced".
- Projeções (classe estática ForeclosureLegalProceedingsProjection):
  * Read: valida schemaVersion, enums e coleções antes de qualquer uso;
  * ForApi: payload público do card (partes e número do processo são registro público e
    fazem parte do contrato visual);
  * ForReport: resumo para o LLM — clerkCoverage, existência de processo, caseTypeCode,
    caseStatus, filingDate, lisPendens, indicadores, noticeTypeCode e data do documento,
    contagem de processos, sourceStatus, limitations. Sem nomes de partes, sem número de
    processo, sem CFN, sem descrição legal, sem texto livre.

---------------------------------------------------------------------------------------------
## 11. BACKEND [ESPECÍFICO DO CARD 8, SEGUINDO 3, 4 E 5]
---------------------------------------------------------------------------------------------
- Application/Contracts/ForeclosureLegalProceedingsContracts.cs: payload tipado + portas
  IRealEstatePropertyDetailClient (GetAsync(propertyId) -> Found | NotFound + data) e
  IMiamiDadeClerkRecordsClient (GetOfficialRecordsByFolioAsync, GetCivilCaseAsync), com DTOs
  próprios e só os campos de 9.2/9.3.
- Application/Contracts/ForeclosureLegalProceedingsProjection.cs: Read, ForApi, ForReport.
- Application/Options/ForeclosureLegalProceedingsOptions.cs: { SourceTimeout=8 s,
  TotalBudget=25 s, MaxPayloadBytes=30.000, MaxClerkRequestsPerRun=2 }; validação no
  NewLegalAnalysisModuleDependencyInjection (SourceTimeout <= TotalBudget; TotalBudget <
  WorkerTimeout; MaxPayloadBytes < limite geral; MaxClerkRequestsPerRun entre 1 e 5),
  conforme 4.1. O orçamento cobre a cadeia Official Records -> Civil (8 s + 8 s).
- Integrations.Infrastructure/RealEstateApi/Clients/RealEstateApiPropertyDetailClient.cs: no
  padrão do RealEstateApiParcelClient (host fixo api.realestateapi.com, x-api-key, limite de
  bytes da resposta, conferência data.id == id pedido). Reutiliza RealEstateApiOptions.
- Integrations.Infrastructure/MiamiDade/Clients/MiamiDadeClerkRecordsClient.cs + options
  MiamiDadeClerkOptions { Enabled, BaseUrl, AuthKey } com ValidateOnStart (Enabled exige
  BaseUrl https no host www2.miamidadeclerk.gov e AuthKey preenchido). Lê Status/StatusDesc do
  envelope antes dos dados e devolve UnitsBalance.
- Registro dos dois clientes com ConfigureEnvironmentalHttpClient no
  IntegrationsDependencyInjection (sem BuildRetryPolicy; sem nova tentativa — cada requisição
  é cobrada). Sem ProjectReference novo (já existe).
- Infrastructure/Workers/ForeclosureLegalProceedingsWorker.cs:
  Id = LegalAnalysisWorkerIds.ForeclosureLegalProceedings ("foreclosure-legal-proceedings");
  registro explícito em AddNewLegalAnalysisWorkers. Sequência: identidade (seção 8) ->
  PropertyDetail e Official Records em paralelo, cada um isolado com timeout -> Civil
  (depende do número do processo) -> montagem pelas regras de 6.1 e 9.3 -> limite de payload.
  Status pela regra 4.1 (seção 10). TimeProvider; ILogger + ILegalAnalysisRunTrace com RunId,
  sourceId, status, durationMs e unitsBalance; nunca folio, número de processo, nome de
  parte, AuthKey, URL ou payload.
- LegalAnalysisReportPromptBuilder: switch por WorkerId (seção 5, item 5) com
  ForeclosureLegalProceedings -> ForeclosureLegalProceedingsProjection.ForReport; demais
  inalterados. O LLM nunca decide status de processo, existência de lis pendens ou indicador.
- LegalAnalysisWorkerPayloadMapper: acrescentar ForeclosureLegalProceedings ->
  ForeclosureLegalProceedingsProjection.ForApi, mantendo os casos existentes.
- Não alterar: coordenador, orquestrador, cesta, clientes MLS/RentCast/Parcel, DTOs MLS,
  RealEstateApiMapper, LeadTypeCuration, persistência, contratos HTTP, outros workers.

---------------------------------------------------------------------------------------------
## 12. FRONTEND (Angular) [ESPECÍFICO DO CARD 8, SEGUINDO 4.2 E 5]
---------------------------------------------------------------------------------------------
- data-access/models/legal-analysis-foreclosure-payload.model.ts (payload) e
  legal-analysis-foreclosure.model.ts (view model).
- legal-analysis-result.model.ts: ForeclosureLegalProceedingsPayload na união de rawPayload.
- legal-analysis-worker-payload.ts: getForeclosureLegalProceedingsPayload (workerId
  'foreclosure-legal-proceedings' + schemaVersion 1), no padrão de
  getFloodEnvironmentalRiskPayload.
- data-access/legal-analysis-foreclosure.mapper.ts + .spec + .fixture: payload -> view model
  (labels, pills, datas, CFN/book-page, rodapé de fontes, omissões).
- ui/legal-analysis/cards/foreclosure-legal-proceedings/: standalone, signals, colapsável
  (seção 5, item 11).
- Layout expandido — a imagem de referência manda (PROMPT, Regra 7):
  1) Cabeçalho: ícone de balança em tile claro + título "Foreclosure & Legal Proceedings" em
     gradiente (grafia da imagem, com "&") + seta à direita. "Layout 2" e o chip
     "Illustrative data" aparecem só no laboratório (input sampleData, como o Summary); no
     card definitivo não existem.
  2) Linha de dois painéis lado a lado (esquerdo ~46%, direito ~54%), cada um com título em
     faixa, linhas "ícone · rótulo em negrito · valor" e divisórias finas:
     esquerdo "Case overview": Case number, Court, Case type, Parties, Case status (pill);
     direito "Filing, recording & property details": Filing date, Lis pendens (pill),
     Recording date, CFN / book-page, Legal description (quebra em várias linhas), Related
     case.
  3) Faixa inteira "Case indicators & document", em uma linha com divisórias verticais:
     título com ícone · Pre-foreclosure (pill) · Auction (pill) · REO (pill) · Document type
     (texto) · Document date (texto). Rótulo em cima, valor embaixo.
  4) Rodapé em itálico: "Source: <fontes que responderam>" + data da consulta + limitações.
- Textos e pills (únicos permitidos):
  * Case status: Open | Closed | Reopened | <texto do Clerk>;
  * Lis pendens: Yes | Released | Not found | Unavailable;
  * Indicadores: Reported | Not confirmed | Unavailable;
  * campo sem dado: "Not available"; Related case sem dado: linha omitida;
  * CFN / book-page: "CFN <ano>R-<sequência> / BK <book> PG <page>"; datas "MMM d, y" (en-US);
  * Parties com kind "lenderOnly": "<lender> (lender reported)";
  * documento histórico: sufixo "(historical record)";
  * com mais de um processo no folio: "+<n> other case(s) on this parcel" ao lado de Related
    case.
- Linha compacta (não está na imagem; proposta para aprovação junto com o HTML estático):
  "Foreclosure & Legal Proceedings · Case <número | not found> · <case type> · <status> ·
  Lis pendens: <Yes | Released | Not found | Unavailable> · Indicators: <n reported | not
  confirmed | unavailable>".
- Responsividade: até 900 px os dois painéis empilham; a faixa de indicadores quebra em
  grade de 3 colunas e, até 600 px, em 1 coluna.
- Rota de teste: features/listings/pages/card-foreclosure (path "card-foreclosure", fora do
  Shell, canMatch !environment.production, loadComponent), no padrão de pages/card-flood.
  Variantes de layout só na rota de teste, todas consumindo o MESMO mock; após aprovação
  permanece apenas a variante aprovada (a imagem recebida é a "Layout 2").
- Composição no diálogo: computed foreclosureCard + bloco @if após o bloco do card de número
  anterior já implementado (Card 7, 6 ou 5) e antes do Summary.
- Regras ESLint de template do AGENTS.md; a11y teclado/leitor de tela (pill com texto, nunca
  só cor; ícones decorativos com aria-hidden). Sem link para documento e sem PDF na fase 1.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE
---------------------------------------------------------------------------------------------
Backend (unit): cada campo de 6.1 com fonte primária, com alternativa e ausente; indicadores
true/false/chave ausente (ausente != false); documento ativo x histórico; lis pendens
recorded/released/notFound/unavailable; agrupamento por instrumento (uma linha por parte);
escolha do registro principal com 0/1/N registros; relatedCases; folio com 13 dígitos, com
máscara, inválido e divergente entre fontes; Miami-Dade / outro condado da FL / condado
ausente / fora da FL; imóvel sem coordenadas (card montado normalmente); sem id do provedor;
Civil consultado pelo número do provedor quando não há registro por folio; "sourcesDiverge";
teto MaxClerkRequestsPerRun; Clerk desabilitado e saldo zerado; orçamento/parcial com
TimeProvider fake; isolamento de fonte (timeout/falha); limite de payload com corte das partes
pesadas; projeções Read/ForApi/ForReport; payload mapper; prompt sem nome de parte, número de
processo, CFN ou descrição legal.
Backend (segurança): AuthKey ausente de log, exceção, payload e telemetria; nenhuma URL do
Clerk gravada; valores financeiros do aviso ausentes do payload.
Backend (reaproveitamento): MLS buscado por SharedMlsQueries (com e sem coordenador no
contexto, como em LegalAnalysisSourceCoordinatorTests); nenhuma chamada direta a
IOnlinePropertyDetail no worker; leitura pela cesta antes de buscar; nenhum uso de
FloodPropertyLocationResolver; nenhuma leitura de public.leadTypes.
Backend (integração): fixtures gravadas dos gates 15.1 e 15.2 (PropertyDetail com e sem
foreclosure; Official Records com lis pendens, com liberação e vazio; Civil aberto e
encerrado); worker no orquestrador sem afetar property-data-cache e Flood; WebApiBoundaryTests
e suíte completa.
Frontend: specs do mapper, do componente (expandido/compacto/parcial/sem dados/fora de
Miami-Dade) e dos guards; nenhum texto "No foreclosure", "Clear" ou "Confirmed" renderizado;
smoke desktop/mobile com capturas no work item, comparadas com a imagem de referência.
Homologação real: imóvel de Miami-Dade com foreclosure aberto e lis pendens; com lis pendens
liberado; sem registro; imóvel da FL fora de Miami-Dade com indicador do provedor; Clerk
indisponível; PropertyDetail indisponível. Conferir no log/LegalAnalysis.txt os eventos Worker
e SourceQuery da execução e o saldo de unidades.
Aceite: todos os campos da imagem dinâmicos e com fonte; cards já implementados e Summary
inalterados; nenhum dado de outro card repetido; nenhum valor simulado; nenhuma ausência
tratada como "sem processo"; no máximo 2 requisições ao Clerk por execução; execução total
< 120 s.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO
---------------------------------------------------------------------------------------------
1) Validação de dados e fontes: este documento + conferência anti-duplicidade + gates 15.1 e
   15.2 (o 15.2 depende de cadastro e formulário notarizado: iniciar já).
2) Rota card-foreclosure com HTML estático nas versões expandida (fiel à imagem) e compacta;
   aprovação visual.
3) Worker com RealEstateAPI PropertyDetail (indicadores, documento, data de registro, número
   do processo publicado, descrição legal alternativa): options, registro no DI, cliente,
   projeções, PromptBuilder, PayloadMapper, modelo/guard/mapper/componente Angular, composição
   no diálogo. Os campos do Clerk saem como "Not available" com a limitação correspondente.
4) Miami-Dade Clerk (Official Records + Civil) como incremento aditivo, quando o 15.2 estiver
   confirmado: só preenche seções do mesmo payload; o card não muda de forma.
5) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile, execução real e leitura do log.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
---------------------------------------------------------------------------------------------
15.1 RealEstateAPI: obter UMA resposta real de /v2/PropertyDetail de um imóvel da Flórida em
     foreclosure e anotar: presença de `reo` e `foreclosure` na raiz; campos reais de
     foreclosureInfo[] (caseNumber, noticeType, lenderName); significado dos códigos de
     noticeType; formatos de data; resposta para id inexistente; custo em créditos; e se o
     plano contratado inclui o endpoint. Sem `reo` na resposta, o indicador REO sai
     "Unavailable". Bloqueia o passo 3.
15.2 Miami-Dade Clerk: criar a conta de Developer, enviar o Access Request Form notarizado,
     comprar unidades e obter o AuthKey; confirmar restrição por IP (o envelope devolve
     IPAddress) a partir do egress de produção. Com chamadas reais, anotar: nomes dos campos
     no JSON (a documentação só mostra o corpo completo em XML); códigos DOC_TYPE de lis
     pendens e de liberação; formato do folio; SE os registros de lis pendens trazem
     FOLIO_NUMBER (cobertura da busca por folio); formato de CASE_NUM aceito por api/Civil;
     valores reais de caseStatus, caseType, courtLocation e juditialSection; a tabela de
     códigos de tribunal no número do processo (CA, CC); códigos de Status/StatusDesc para
     "sem resultado", erro e saldo insuficiente; limite de taxa.
     Refazer a conferência depois de 2026-12-01 (nova versão anunciada). Bloqueia o passo 4.
15.3 Decidir: publicar o passo 3 (só provedor) antes do Clerk. Recomendação: sim — entrega os
     indicadores, o documento e a data com estado honesto nos demais campos, e o Clerk entra
     sem mudar o payload nem o layout.
15.4 Decisão de produto: exibir indicadores de foreclosure e nomes das partes no relatório
     Legal Analysis, mantendo a curadoria #357 na página do imóvel; confirmar que nomes de
     partes (registro público) podem ir para o pacote de auditoria no Blob e por quanto tempo;
     confirmar nos termos da RealEstateAPI o armazenamento desses campos no pacote.
15.5 Fronteira com o Card 10: registrar no blueprint do Card 10 que o lis pendens pertence ao
     Card 8 (6.3) e decidir lá se a descrição legal da escritura será exibida. Se o Card 6
     fase 2 usar api/Civil, a fonte vira compartilhada (3.2).
15.6 Imagem de referência: salvar em legal-share (padrão CARD8_FORECLOSURE_IMG_REF.<ext>);
     aprovar a linha compacta da seção 12; informar se existem outras variantes além da
     "Layout 2".
15.7 Custo por execução a aprovar: 1 crédito RealEstateAPI + até US$ 0,40 do Clerk.
15.8 Uso de LLM no card: em aberto (arquiteto Bruno, PROMPT 8E). O worker entrega fatos; o LLM
     só recebe ForReport.
15.9 Base de código (reconferido em 2026-11-10): d_leevli-platform-api está em
     feat/legal-analysis-source-coordinator (57074dd6 "Modulariza Workers"), que contém o
     Card 5 e o Plano A; d_leevli-platform-web está em LEGAL (71e79913). Os Cards 6 e 7 não
     estão implementados: as peças compartilhadas (switch do PromptBuilder, helper de MLS de
     3.3) são criadas por quem for implementado primeiro; o outro só acrescenta. A branch do
     card parte da branch de integração que contenha Card 5 + Plano A.
15.10 Work item com critérios de aceite; branch feat/<card>-foreclosure-legal-proceedings;
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
