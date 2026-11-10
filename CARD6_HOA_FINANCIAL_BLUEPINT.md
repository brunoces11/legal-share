# LEEVLI | LEGAL ANALYSIS | CARD 6 — HOA FINANCIAL EXPOSURE
# MASTER IMPLEMENTATION BLUEPRINT V3 — TEMPLATE PARA OS CARDS 7, 8, 9...
# (aderente ao código em 2026-11-06, incluindo o Plano A: coordenador de consultas por execução)

Este documento tem duas funções:
1. Blueprint de implementação do Card 6 (HOA Financial Exposure).
2. Template técnico para os próximos cards. Para criar o Card N, copie a estrutura DO
   DOCUMENTO, mantenha integralmente as seções marcadas como [REGRA FIXA] e substitua as
   seções marcadas como [ESPECÍFICO DO CARD] pelo conteúdo do novo card. O template padroniza
   o documento e o encaixe no pipeline; não manda copiar o comportamento de outro card.

V3 (2026-11-06): seções fixas alinhadas com o Card 7 (princípio "comum x de cada card",
identidade própria de cada card, status do worker, registro único de estatutos); Card 6 deixa
de usar o resolvedor de localização do Card 5 (exige coordenadas).

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
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 6]
---------------------------------------------------------------------------------------------
Card único e consolidado da associação (HOA/COA): todo dado financeiro útil aparece uma única
vez, vindo de fonte verificável — API estruturada OU documento/dataset público estável com URL.
A origem (API x documento) é só detalhe arquitetural; para o usuário vale: útil, relevante,
confiável, implementável. Variantes de layout mostram exatamente o mesmo view model.

Fase 1 (este blueprint, 1 PR/1 card), conforme PROMPT_DEV_LEGAL.txt e o CSV
Fontes_API_Legal_analyses.csv:
- "Existência de HOA ou associação" -> MLS (RealEstateAPI) + Florida DBPR (condomínios).
- "Taxa mensal da associação" -> MLS: valor, frequência, 2ª associação, itens incluídos
  ("as listed"; CSV: valor no MLS pode estar desatualizado).
- "SIRS e reserva estrutural" -> Florida DBPR SIRS reporting (somente condomínio; só o status
  de reporte, nunca valores de reserva).
- Base legal financeira -> Florida Statutes oficiais (Online Sunshine), por regime.
Fase 2 (backlog, card/PR próprio após homologação de cada fonte): "Special assessment",
"Saúde financeira da associação" (budget, reservas, inadimplência, dívidas), "Seguro master do
prédio", estoppel (valores e cobranças da unidade), histórico de taxas, "Litígios da
associação" (County Clerk Civil Dockets, por condado; o CSV só cita Miami-Dade, a jurisdição
deve ser resolvida antes), RentCast property records (hoa.fee) como fonte cruzada. Motivo:
dependem de documentos privados da associação (sem URL pública permanente) ou de fonte não
integrada; não há fluxo de recebimento de documentos no NewLegalAnalysis (mesma lacuna dos
Cards 7 e 15; o fluxo será único para os três).

Fronteira com outros cards (regra anti-duplicidade, ⚠️ item 4):
- Fora do Card 6: regras de uso/locação/pets/reforma/manutenção (Card 7), Milestone
  Inspection (Card 9), liens de condomínio/HOA §718.116/§720.3085 registrados (Card 10), CDD
  (Card 11), FHA Condo (Card 14).
- Conferido em 2026-11-06: nenhum card atual (1, 3, 4, 5) exibe taxa, nome ou regime da
  associação. O Card 6 é o DONO desses dados no relatório.
- O Card 7 não exibe nome, regime nem registro da associação e não consulta o DBPR: ele
  referencia o Card 6 (indicador "see Card 6").

Proibições invariantes: sem HOA Health Score, "Low Risk", gauge, pesos, % de funding, saldo de
reservas ou qualquer número não retornado por fonte; sem placeholder em produção; null nunca
vira 0; taxa ausente != "sem HOA"; ausência de SIRS na lista DBPR != estudo inexistente ou
dispensado; valor MLS sempre rotulado "as listed" com data do anúncio; responsável por
pagamento nunca presumido.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM EM 2026-11-06) [ESPECÍFICO DO CARD 6]
---------------------------------------------------------------------------------------------
- mlsSearchDetail (RealEstateApiMlsListingItemDto -> listing.property = RealEstateApiPropertyDto):
  contém AssociationFee (decimal?) — único VALOR de taxa disponível hoje. Sem nome,
  2ª associação ou itens incluídos.
- mlsDetail (RealEstateApiMlsDetailDataDto, /v2/MLSDetail): a classe tipada não declara campo
  HOA, mas HomeDetails (JsonElement) é preservado no cache e serializado para o payload do
  property-data-cache. As gravações reais usadas em RealEstateApiOnlinePropertyDetailTests.cs
  (listings A12004619 e A12016406) mostram data.homedetails.associationFeeFrequency (nome
  CONFIRMADO; valor null nas duas gravações, nunca visto preenchido). Valor e frequência vêm
  de respostas diferentes (mlsSearchDetail x mlsDetail): o worker lê as duas e rotula a origem.
- Pela chave detailId, IOnlinePropertyDetail.GetAsync também grava em cache o item de
  mlsSearchDetail quando o registro tem listingId (enriquecimento interno). Para lê-lo, usar o
  listingId do próprio registro (o FloodPropertyLocationResolver faz isso com
  expectedListingId, linhas 63–68; usar como referência, não chamar o resolvedor).
- FloodPropertyLocationResolver.ResolveIdentityAsync devolve null sem latitude/longitude
  válidas (linhas 55–57) e não devolve ZIP nem subdivisão: não serve ao Card 6 (seção 8).
- Seção técnica do diálogo já lê 'associationFee' (PROPERTY_FIELDS, "Association fee") do
  payload do property-data-cache. Isso é detalhamento técnico, não um card do relatório.
- RealEstateAPI MLS Search v3 documenta filtros has_hoa, hoa_amenities, hoa_association_name,
  hoa_association_name2, hoa_fee_min/max, hoa_fee2_min/max, hoa_fee_frequency,
  hoa_fee_frequency2 (docs lidas em 2026-11-02). São filtros de request: indicam que o dataset
  tem esses atributos, mas os NOMES no response não estão confirmados -> gate 15.1.
- RentCast: IRentCastClient só tem GetAvmAsync, GetRentEstimateAsync e
  GetMarketStatsByZipAsync; /v1/properties (hoa.fee, "HOA Fee Amount (Monthly)", documentado)
  NÃO integrado. O AVM devolve null tanto em falha quanto sem preço (não elegível ao
  coordenador sem adaptação).
- DBPR: nenhum cliente no codebase. Páginas do CSV retornaram 403 ao fetch automatizado
  -> gate 15.2.
- Divergências FORA do escopo (não corrigir neste PR, reportar ao arquiteto):
  PropertyDetailDto.Hoa (decimal?) sem mapeamento de origem; property-detail.mapper.ts tipa
  hoa como boolean; building-overview-section usa default estático ('$385/mo') na página do
  imóvel.

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE [ESPECÍFICO DO CARD 6, SEGUINDO 3.3]
---------------------------------------------------------------------------------------------
- Resolução própria do Card 6, pelos mecanismos obrigatórios de 3.3: PropertyLookup +
  IPropertyDataCacheReader; com FetchMissingSources=true, MLS ausente buscado SOMENTE por
  SharedMlsQueries (DetailAsync/ListingAsync), compartilhando a consulta com os demais workers.
- No Card 6 o registro MLS é ao mesmo tempo pré-requisito (identidade para o match DBPR) e
  fonte de conteúdo (taxa e frequência).
- Campos lidos: endereço, unidade, city, state, ZIP, county, propertyType/subType, subdivisão
  (SubdivisionName), nome(s) da associação quando o gate 15.1 confirmar. Coordenadas NÃO são
  necessárias: imóvel sem latitude/longitude continua com o card montado.
- Não usar FloodPropertyLocationResolver (3.3).
- Regime da associação: "condominium" só com match DBPR inequívoco; caso contrário
  "unconfirmed". Townhouse/propertyType NÃO define regime (CSV). Fora da Flórida: DBPR e
  estatutos FL -> notApplicable; MLS continua. O Card 6 é o único card que determina o regime.

---------------------------------------------------------------------------------------------
## 9. FONTES DA FASE 1 [ESPECÍFICO DO CARD 6]
---------------------------------------------------------------------------------------------
### 9.1 RealEstateAPI MLS (primária para cobranças) — fonte COMPARTILHADA
- Leitura pela cesta, sem alterar nenhum DTO: valor = mlsSearchDetail.listing.property.
  associationFee; frequência = mlsDetail.homeDetails.associationFeeFrequency (leitura
  tolerante de JsonElement com PropertyLookup.FindObject/FindString; texto/null).
- Busca quando ausente: só via SharedMlsQueries (3.2). Proibido chamar IOnlinePropertyDetail
  direto.
- Nome(s) da associação, 2ª associação e itens incluídos: só entram após o gate 15.1 (declarar
  nos DTOs MLS apenas campos confirmados no response real, nullable e aditivo; afeta
  Properties/Integrations e o formato da cesta: rodar os testes desses módulos e do
  NewLegalAnalysis).
- Regras: frequência ausente (inclusive null, o caso das duas gravações) -> exibir valor com
  "frequency not reported" e NÃO anualizar; anualizado = valor x fator da frequência publicada,
  usando somente os valores confirmados no gate 15.1 (esperados: Monthly 12, Quarterly 4,
  Semi-Annually 2, Annually 1); qualquer outro valor -> não calcular, exibir original.
  Segunda associação exibida separada, nunca somada sem frequência confirmada das duas.
  "Fee includes" exibe valores originais do provedor, sem tradução por suposição.
  Data de referência = ModificationTimestamp/ListingContractDate do anúncio.
### 9.2 Florida DBPR — registro de condomínios — fonte EXCLUSIVA do Card 6 (cliente direto)
- Gate 15.2 antes do código: confirmar URL do dataset público, formato, colunas (nome da
  associação, nº do projeto/arquivo, condado, unidades, administradora, endereço, status) e
  periodicidade, a partir do egress de produção.
- Match: nome da associação (MLS) + condado + endereço/ZIP normalizados. 0 candidatos ->
  noResults ("No matching DBPR condominium record"; não significa "não é condomínio");
  >1 sem desempate -> ambiguous, exibir só a contagem; nunca escolher arbitrariamente.
- Se for arquivo bulk: espelho local versionado (snapshot com retrievedAtUtc, URL de origem e
  hash), carregado em memória com limite; atualização manual na fase 1, rotina de verificação
  na fase 2. Nunca baixar o arquivo por execução.
- O Card 7 não usa o DBPR. Se outro card vier a usá-lo, a fonte vira compartilhada: aplicar
  3.2 (chave em LegalAnalysisSourceKeys, contrato que separa falha de "sem resultados").
### 9.3 Florida DBPR — SIRS reporting (condomínio confirmado em 9.2)
- Mesmo gate 15.2: lista pública de reportes SIRS (formato/campos/atualização).
- Exibir: "SIRS completion reported to DBPR" + data quando houver registro; sem registro ->
  "No SIRS report found in the DBPR list" + limitação (não prova ausência/dispensa). Nunca
  exibir valores de reserva/funding (CSV: só quando divulgados no estudo -> fase 2).
### 9.4 Florida Statutes (referências legais financeiras) — sem chamada externa
- Entradas do Card 6 no registro único de estatutos (3.4). Confirmadas em 2026-11-02:
  §718.116 "Assessments; liability; lien and priority..."; §720.30851 "Estoppel
  certificates"; §718.112 "Bylaws" (subseções (2)(f)-(g) budget, reservas e SIRS); §720.303
  "Association powers and duties; ... official records; budgets; financial reporting...",
  citada pelas subseções (4)-(7) (CSV "Saúde financeira da associação"). A subseção
  §720.303(1) é exibida pelo Card 7 (manutenção); se a fase 2 de litígios precisar dela
  (CSV "Litígios": §720.303(1),(4)-(5)), referencia o Card 7.
  Incluir também §718.111(11)-(13) (seguro, registros, relatório financeiro) após conferir
  heading/URL.
- Regime condominium -> cap. 718; regime unconfirmed -> 718 e 720 com rótulo "applies
  depending on the association's legal regime". Texto exibido: só heading oficial + link; sem
  interpretação jurídica nem conclusão sobre o imóvel.
- Ponte para a fase 2: cada estatuto lista o documento que o rege (estoppel, budget, financial
  report, SIRS/reserve study, insurance), como "Documents governed by this section", sem
  afirmar que existem ou que o comprador tem direito a eles.

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD (worker "hoa-financial-exposure", schemaVersion 1) [ESPECÍFICO]
---------------------------------------------------------------------------------------------
{
  schemaVersion, collectedAtUtc,
  association: { status, regime:{value:"condominium"|"unconfirmed", basis},
                 names[]:{name, role:"primary"|"secondary", sourceId} },
  charges: { status, asOf?, sourceId,
             items[]:{ role, amount?{value,currency:"USD"}, frequencyRaw?, annualized?,
                       includesRaw[] }, limitations[] },
  condominiumRegistry: { status, freshness{retrievedAtUtc, datasetUrl},
                         match?{projectNumber?, associationName, county?, units?,
                                managingEntity?}, candidatesCount? },
  sirs: { status, reportedAt?, freshness, datasetUrl? },
  legalReferences[]: { section, heading, url, statuteYear,
                       regime:"condominium"|"hoa"|"general", governs[] },
  notCoveredByPublicSources[]: códigos fixos (specialAssessments, reserves, budget,
                               delinquency, masterInsurance, litigation, estoppel),
  sourceStatus[]: formato padrão (4.1),
  limitations[]
}
- status das seções: valores de FloodSourceStatus (4.1). Atualidade separada do status.
- Sem payload bruto do provedor; payload alvo <= 30 KB (HoaFinancialExposureOptions.
  MaxPayloadBytes menor que o limite geral).
- Projeções (classe estática HoaFinancialExposureProjection):
  * Read: valida schemaVersion e coleções antes de qualquer uso;
  * ForApi: payload público do card (por desenho, sem campos internos);
  * ForReport: resumo para o LLM — regime, charges (valor, frequência, asOf), status DBPR e
    SIRS, sourceStatus, limitations. Sem nomes de pessoas, sem endereço, sem texto livre.
- Fatos de identidade já exibidos por outros cards (endereço, tipo do imóvel) NÃO entram no
  payload de exposição (⚠️ item 4); são usados só para o match.

---------------------------------------------------------------------------------------------
## 11. BACKEND [ESPECÍFICO DO CARD 6, SEGUINDO 3, 4 E 5]
---------------------------------------------------------------------------------------------
- Application/Contracts/HoaFinancialExposureContracts.cs: payload tipado + porta
  IFloridaDbprCondominiumClient (registro + SIRS, DTOs próprios).
- Application/Contracts/HoaFinancialExposureProjection.cs: Read, ForApi, ForReport.
- Application/Options/HoaFinancialExposureOptions.cs: { SourceTimeout=8 s, TotalBudget=20 s,
  MaxPayloadBytes=30.000 }; validação no NewLegalAnalysisModuleDependencyInjection
  (SourceTimeout <= TotalBudget; TotalBudget < WorkerTimeout; MaxPayloadBytes < limite
  geral), conforme 4.1.
- Integrations.Infrastructure/FloridaDbpr/Clients: implementação; options com
  ValidateOnStart e registro com ConfigureEnvironmentalHttpClient no
  IntegrationsDependencyInjection (sem BuildRetryPolicy; no máximo 1 nova tentativa em
  5xx/timeout). Sem ProjectReference novo (já existe).
- Registro de estatutos: entradas do Card 6 no registro único (3.4); se ainda não existir,
  o Card 6 o cria.
- Infrastructure/Workers/HoaFinancialExposureWorker.cs:
  Id = LegalAnalysisWorkerIds.HoaFinancialExposure ("hoa-financial-exposure"); registro
  explícito em AddNewLegalAnalysisWorkers. Fontes isoladas, identidade própria (seção 8) com
  MLS pela cesta + SharedMlsQueries, DBPR pelo cliente direto, estatutos pelo registro único.
  Status do worker pela regra 4.1. TimeProvider; ILogger + ILegalAnalysisRunTrace sem
  endereço, nome de associação ou payload.
- LegalAnalysisReportPromptBuilder: switch por WorkerId (seção 5, item 5) com
  HoaFinancialExposure -> HoaFinancialExposureProjection.ForReport; demais inalterados.
  O LLM nunca calcula anualizado nem decide regime.
- LegalAnalysisWorkerPayloadMapper: acrescentar HoaFinancialExposure ->
  HoaFinancialExposureProjection.ForApi, mantendo os casos existentes.
- Não alterar: coordenador, orquestrador, cesta, clientes MLS/RentCast, persistência,
  contratos HTTP, outros workers.

---------------------------------------------------------------------------------------------
## 12. FRONTEND (Angular) [ESPECÍFICO DO CARD 6, SEGUINDO 4.2 E 5]
---------------------------------------------------------------------------------------------
- data-access/models/legal-analysis-hoa-financial-payload.model.ts (payload) e
  legal-analysis-hoa-financial.model.ts (view model).
- legal-analysis-result.model.ts: HoaFinancialExposurePayload na união de rawPayload
  (hoje PropertyDataCacheWorkerPayload | FloodEnvironmentalRiskPayload).
- legal-analysis-worker-payload.ts: getHoaFinancialExposurePayload (workerId
  'hoa-financial-exposure' + schemaVersion 1), no padrão de getFloodEnvironmentalRiskPayload.
- data-access/legal-analysis-hoa-financial.mapper.ts + .spec + .fixture: payload -> view model
  (labels, moeda, frequência, omissões).
- ui/legal-analysis/cards/hoa-financial-exposure/: standalone, signals, colapsável (seção 5,
  item 11).
- Layout expandido (um bloco por dado, sem repetição):
  1) Cabeçalho: título + nome(s) da associação + regime (ou "Association regime unconfirmed").
  2) Painel "Financial Snapshot" (somente fatos): taxa principal "as listed", frequência,
     anualizado (se calculável), 2ª associação, status SIRS. Sem nota/score.
  3) Fee details: includes, data do anúncio.
  4) Association registry (DBPR): projeto, condado, unidades, administradora.
  5) Florida law references: heading + link oficial + "Documents governed".
  6) Rodapé: "Not covered by public sources" (lista fixa) + fontes/datas/limitações.
- Linha compacta: "HOA Financial Exposure · $<fee> <frequência> (as listed) ·
  <associação ou 'Association unconfirmed'> · SIRS: <reported date | not found | n/a>".
- Rota de teste: features/listings/pages/card-hoa-fin (path "card-hoa-fin", fora do Shell,
  canMatch !environment.production, loadComponent), no padrão de pages/card-flood. Variantes
  de layout só na rota de teste, todas consumindo o MESMO mock; após aprovação permanece
  apenas a variante aprovada.
- Composição no diálogo: computed hoaFinancialCard + bloco @if após o bloco do Card 5 e antes
  do Summary.
- Regras ESLint de template do AGENTS.md; a11y teclado/leitor de tela. Sem PDF na fase 1.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE
---------------------------------------------------------------------------------------------
Backend (unit): fee com/sem frequência, frequências não mapeadas, 2ª associação, null != 0,
regime por match DBPR (0/1/N candidatos), SIRS found/noResults/notApplicable, fora da FL, sem
MLS, imóvel sem coordenadas (card montado normalmente), orçamento/parcial com TimeProvider
fake, isolamento de fonte (timeout/falha), limite de payload, entradas do registro de
estatutos por regime, projeções Read/ForApi/ForReport, payload mapper, prompt sem dados
sensíveis.
Backend (reaproveitamento): MLS buscado por SharedMlsQueries (com e sem coordenador no
contexto, como em LegalAnalysisSourceCoordinatorTests); nenhuma chamada direta a
IOnlinePropertyDetail no worker; leitura pela cesta antes de buscar; nenhum uso de
FloodPropertyLocationResolver; registro de estatutos único.
Backend (integração): fixtures gravadas (MLS condo com HOA, DBPR, SIRS); worker no orquestrador
sem afetar property-data-cache e Flood; DTOs MLS estendidos sem quebrar Properties/Integrations;
WebApiBoundaryTests e suíte completa.
Frontend: specs do mapper, componente (expandido/compacto/parcial/sem dados) e guards; smoke
desktop/mobile com capturas no work item.
Homologação real: condomínio em Miami-Dade com HOA no MLS e registro DBPR; casa em HOA (regime
unconfirmed); imóvel sem taxa no MLS; DBPR indisponível; caso ambíguo. Conferir no
log/LegalAnalysis.txt os eventos Worker e SourceQuery da execução.
Aceite: cards 1–5 e Summary inalterados; nenhum dado de outro card repetido; nenhum valor
simulado; todo dado com fonte/data; nenhuma ausência tratada como "sem associação/sem
cobrança"; execução total < 120 s.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO
---------------------------------------------------------------------------------------------
1) Validação de dados e fontes: este documento + conferência anti-duplicidade + gates 15.1/15.2.
2) Rota card-hoa-fin com HTML estático nas versões expandida e compacta, usando só campos que
   a fase 1 entrega; aprovação visual.
3) Worker com MLS (valor + frequência, via cesta + SharedMlsQueries) + registro de estatutos:
   entrega útil sem integração nova; options, registro no DI, projeções, PromptBuilder,
   PayloadMapper, modelo/guard/mapper/componente Angular, composição no diálogo.
4) DBPR (registro + SIRS) como incremento aditivo, quando o 15.2 estiver confirmado: só
   acrescenta seções ao mesmo payload; o card não muda de forma.
5) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile, execução real e leitura do log.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
---------------------------------------------------------------------------------------------
15.1 Conferência de campos (feita pela equipe antes de codar): obter UMA resposta real de
     MLSDetail/MLSSearch de um condomínio com HOA e anotar os nomes exatos dos campos de nome
     da associação, 2ª associação, itens incluídos e os valores reais de frequência. Sem isso,
     a fase 1 sai só com valor + frequência (associationFee + homeDetails.
     associationFeeFrequency); o restante fica omitido. Não bloqueia o início (14, passos 2–3).
15.2 Confirmar, do egress de produção, dataset DBPR de condomínios e lista SIRS (fetch
     automatizado recebeu 403). Bloqueia 9.2/9.3; sem eles o card sai só com MLS + estatutos.
15.3 Decidir: espelho local do DBPR (local de armazenamento e rotina manual) vs consulta
     online, conforme o formato confirmado.
15.4 Decidir: integrar RentCast /v1/properties (hoa.fee mensal) na fase 1 como fonte cruzada
     (custo por chamada) ou manter na fase 2. Se integrar, separar falha de "sem resultados"
     no contrato (3.2).
15.5 Uso de LLM no card: em aberto (arquiteto Bruno). O worker entrega fatos; o LLM só recebe
     ForReport.
15.6 Base de código (conferido em 2026-11-06 no repositório d_leevli-platform-api): o
     Plano A está commitado (57074dd6 "Modulariza Workers") na branch
     feat/legal-analysis-source-coordinator, publicada no remoto github e criada sobre a
     LEGAL local (1fd41f91), que contém o Card 5. Nem develop nem LEGAL contêm o
     coordenador; develop também não contém o Card 5; a LEGAL local difere de azure/LEGAL.
     A branch do card deve partir da branch de integração que contenha Card 5 + Plano A,
     depois do merge do Plano A.
15.7 Work item com critérios de aceite; branch feat/<card>-hoa-financial-exposure;
     commits feat(new-legal-analysis): #<card> ...
15.8 Observação de fonte: as gravações de referência citam Central HQ
     (20-ARQUITETURA/APIs/responses/realestateapi/02-mlsdetail.json), fora deste repositório;
     não foi lida nesta análise.

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
