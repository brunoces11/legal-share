# LEEVLI | LEGAL ANALYSIS | CARD 13 — PERMITTED USE AND ZONING
# MASTER IMPLEMENTATION BLUEPRINT V1 — BASEADO NO TEMPLATE DO CARD 6 (CARD6_HOA_FINANCIAL.md, V3)
# (aderente ao código em 2026-11-10, incluindo o Plano A: coordenador de consultas por execução)

Este documento é o blueprint de implementação do Card 13 (Permitted Use and Zoning).
Segue a estrutura do template do Card 6: as seções [REGRA FIXA] são idênticas às do template;
as seções [ESPECÍFICO DO CARD] são próprias do Card 13.

V1 (2026-11-10): a IMAGEM DE REFERÊNCIA do card é o contrato visual. Ela traz a versão
expandida e a compacta. Todo dado exibido está mapeado, na seção 6.1, para um campo real de
fonte programática conferida em 2026-11-10: camadas públicas de Miami-Dade County no ArcGIS
(zoning, tabela de parâmetros do distrito, CDMP, municípios e audiências de zoning, com
amostras reais consultadas) e o esquema Standardized Zoning da Regrid. As seções [REGRA FIXA]
foram reconferidas contra o código em 2026-11-10: as referências de arquivo e linha datadas
de 2026-11-06 continuam válidas.

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
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 13]
---------------------------------------------------------------------------------------------
Card do ZONEAMENTO PÚBLICO do lote: jurisdição, distrito de zoning, descrição do distrito,
categoria de uso do solo do plano diretor (CDMP), usos e parâmetros do DISTRITO e o histórico
de processos e resoluções de zoning do lote. Todo dado vem de fonte programática e carrega a
origem. O card informa o que as camadas públicas dizem sobre o distrito; não decide se um uso
pretendido é permitido no imóvel.

Termos do card:
- Distrito ........... zona de zoning em que o ponto do imóvel cai (ex.: RU-2).
- Parâmetro do distrito  controle numérico da norma do distrito (altura, densidade, lote
                       mínimo, recuo). Vale para o distrito, não é aprovação do lote.
- CDMP ............... Comprehensive Development Master Plan de Miami-Dade: mapa de uso do
                       solo futuro do condado. Camada distinta do zoning (CSV: "zoning e
                       future land use são camadas distintas").
- Processo de zoning . pedido registrado no sistema EnerGov do condado (audiência, ajuste
                       administrativo, variance etc.), com resolução quando houver.
- Folio .............. número do parcel em Miami-Dade (13 dígitos). Chave de consulta; não é
                       exibido (dono: Card 1).

O CSV mapeia dois itens para este conteúdo: "Zoneamento do imóvel" (distrito, Future Land
Use, histórico/resolução, aplicações e audiências; fonte "Regrid; Miami-Dade/município
planning & zoning") e "Compatibilidade de uso" (classificação do parcel, approvals,
conditions, variances). Base legal citada: F.S. §163.3194; Miami-Dade Code cap. 33; Broward
Code cap. 39.

### 6.1 CONTRATO VISUAL — IMAGEM DE REFERÊNCIA -> DADOS
Imagem recebida em 2026-11-10 com a versão expandida e a compacta. São marcas do laboratório,
não dados (seção 12): o chip "Illustrative data", o chip "Illustrative record" acima da
tabela, "(example)" após a jurisdição e "• Illustrative example" no rodapé.
Formato de cada linha: "campo (exemplo da imagem) <- fonte primária | alternativa | estado
quando ausente".
Base das fontes de Miami-Dade (ArcGIS hospedado do condado, sem chave):
https://services.arcgis.com/8Pc9XBTAsYuxx9Ny/arcgis/rest/services/<serviço>/FeatureServer/0

Cabeçalho
- Ícone de mapa com pin em tile claro + divisória + título "Permitted Use & Zoning"
  (grafia da imagem, com "&") em gradiente + seta. Sem dado.

Faixa superior (4 itens com ícone, separados por divisórias verticais)
- Jurisdiction (Miami-Dade County)
    <- Municipalitypoly_gdb, ponto do imóvel dentro do polígono: NAME (ex.: "MIAMI SHORES";
       "UNINCORPORATED MIAMI-DADE" -> "Unincorporated Miami-Dade County"), exibido em title
       case
    |  Regrid municipality_name  |  "Not available"
- Zoning district (R-2)
    <- não incorporado: ZonepolyU_gdb.ZONE; município: MunicipalZone_gdb.ZONE (ponto dentro
       do polígono)
    |  Regrid zoning  |  "Not available"
- District description (Residential)
    <- a descrição oficial do MESMO polígono: ZonepolyU_gdb.ZONE_DESC (ex. real: "Two-family
       Residential District, 7,500 ft2 net") ou MunicipalZone_gdb.ZONEDESC, exibida como
       publicada
    |  Regrid zoning_description  |  "Not available"
- CDMP land-use category (Low-medium density residential)
    <- Landuse_gdb.DESCRIPTION, ponto dentro do polígono (camada do CDMP; espelho hospedado de
       LandManagement/MD_CDMP/MapServer/7 "Comprehensive Development Master Plan (CDMP)").
       Ex. real: "LOW-MEDIUM DENSITY RESIDENTIAL (LMDR) 6-13 DU/AC", exibido em sentence case
       sem perder sigla e faixa: "Low-medium density residential (LMDR) 6-13 DU/AC".
    |  fora de Miami-Dade -> "Not applicable" (o CDMP é do condado)  |  "Not available"
    Dentro de município, o plano municipal (MunicipalFutureLandUse_gdb.FLUDESC) é a camada
    que rege -> decisão 15.3.

Bloco A — "District uses" (painel esquerdo, parte de cima)
- Permitted by right (Residential)
    <- Regrid permitted_land_uses_as_of_right  |  sem alternativa (as camadas do condado não
       publicam usos)  |  "Not available"
- Conditional uses (Community facility)
    <- Regrid permitted_land_uses_conditional  |  sem alternativa  |  "Not available"
    Os dois campos são texto do provedor (o esquema descreve permitted_land_uses como "JSON
    object containing lists of use types"); exibir os itens da lista separados por vírgula,
    sem tradução. Formato real -> gate 15.1.

Bloco B — "District parameters" (painel esquerdo, parte de baixo)
- Maximum height (35 ft)
    <- Regrid max_building_height_ft (pés)
    |  condado: MAXHEIGHT (TBLUnincorporatedZoneDistrict pelo ZONE, ou MunicipalZone_gdb).
       A amostra real dá RU-1 = 2: o valor é em PAVIMENTOS, não em pés -> exibido como
       "<n> stories", nunca convertido em pés (unidade confirmada no gate 15.2)
    |  "Not available"
- Maximum density (12 units / acre)
    <- condado: DENSITY (TBLUnincorporatedZoneDistrict ou MunicipalZone_gdb; ex. real RU-1 =
       5.2)  |  Regrid max_density_du_per_acre  |  "Not available"
- Minimum lot area (5,000 sq ft)
    <- condado: MINLOTSIZE (ex. real RU-1 = 7500)  |  Regrid min_lot_area_sq_ft
    |  "Not available"
- Minimum front setback (20 ft)
    <- Regrid min_front_setback_ft  |  sem alternativa (o condado não publica recuos)
    |  "Not available"
- Valores especiais da Regrid (documentados): -5555 -> "Refer to the zoning code";
  -9999 -> "Not applicable". Não são null nem zero.
- Nota fixa em itálico (texto da imagem): "District-level data; property-specific approval
  not confirmed."

Bloco C — "Zoning applications & resolutions" (painel direito)
- Tabela com as colunas da imagem, uma linha por processo do lote:
  * Process (Z2025-00123) <- PROC_NUM (ex. real "Z2025000063"; exibido como publicado)
  * Case type (Zoning adjustment) <- CASE_TYPE (ex. reais "ZM Admin Adjustment",
    "ZM Other Public Hearings"; exibido como publicado)
  * Application (May 2025) <- APPL_MONTH + APPL_YEAR (ex. real "3" + "2025" -> "Mar 2025")
  * Resolution reference (Z-2025-08) <- LAST_RESNO (ex. real "CZAB11-10-26"); vazio -> "—"
  * Record ("View source") <- APPL_LINK, link para o processo no EnerGov do condado (ex.
    real https://energov.miamidade.gov/EnerGov_Prod/SelfService/#/plan/<id>). Só é exibido
    se for https no host energov.miamidade.gov; senão a célula fica "—".
  Fonte: VLANDMANAGEMENTHEARING_gdb (espelho hospedado de
  LandManagement/MD_ZoningRecords/MapServer/0 "Zoning Hearings").
  Busca: FOLIO = folio do imóvel, unida à busca do ponto dentro do polígono do processo;
  deduplicada por PROC_NUM; ordenada da aplicação mais recente para a mais antiga; até 10
  linhas, com "+<n> more" quando houver mais.
- Sem processo encontrado: tabela substituída por "No zoning applications found for this
  parcel in county records." (não significa que não existam processos municipais).
- Fora de Miami-Dade: "Zoning records are not integrated for this county."

Rodapé
- Linha 1, esquerda: "Sources: <fontes que responderam>" com os nomes da imagem: "Miami-Dade
  zoning layers" e "Regrid standardized zoning". Texto fixo é proibido.
- Linha 1, direita: "Source data date: <mês ano>" <- data mais ANTIGA entre as fontes que
  contribuíram: Regrid zoning_data_date (o próprio provedor avisa que é a data de
  processamento, não a data da norma) e a data de última edição da camada do condado usada
  (editingInfo.dataLastEditDate do serviço).
- Linha 2, itálico, fixa (texto da imagem): "Coverage and access vary. Local rules and
  property-specific approvals require confirmation."

Versão compacta (está na imagem)
- Ícone + título | ícone de camadas + distrito ("R-2") | ícone de casa + descrição
  ("Residential", mesmo valor do expandido, truncado com reticências e title com o texto
  completo) | ícone de folha + "CDMP: <categoria>" (texto antes do parêntese, truncado) |
  ícone de documento + "<n> zoning record(s)" (ou "No zoning records" / "Records
  unavailable") + seta para baixo. Ausência -> "Unavailable" no segmento.

Regra de leitura do contrato: cada campo sai com um único sourceId. Precedência por campo
conforme acima. A Regrid só preenche parâmetros e usos se o seu zoning for IGUAL ao distrito
do condado (comparação sem espaços e sem diferença de caixa); se divergir, nenhum dado da
Regrid entra nos blocos A e B ("zoningCodeMismatch"), porque descreveria outro distrito.

### 6.2 Fases
Fase 1 (este blueprint, 1 PR/1 card) = exatamente a imagem:
- Miami-Dade ArcGIS (público, sem chave): jurisdição, distrito, descrição, parâmetros do
  condado, CDMP e processos de zoning.
- Regrid Standardized Zoning (contratada, nacional): usos permitidos/condicionais, recuo
  frontal, altura em pés e alternativa dos demais campos; é a única fonte fora de Miami-Dade.
Fase 2 (backlog, card/PR próprio após homologação e nova imagem):
- "Restrições de locação pública" (CSV; PROMPT 13B): F.S. §509.032(7), Miami-Dade §33-28 e
  ordinances municipais. A imagem não as mostra — ver 15.6 (o Card 7 aponta "see Card 13").
- Overlays (ZonepolyU_gdb.OVLY e zoning_type "Overlay" da Regrid), link da norma
  (zoning_code_link), status do processo (APPL_STATUS em LandManagementHearingView), board.
- Processos de zoning de municípios e de outros condados (Broward primeiro).

### 6.3 Fronteira com outros cards (regra anti-duplicidade, ⚠️ item 4)
- Card 1: endereço, folio, tipo do imóvel (uso fiscal) e proprietário. O Card 13 usa o folio
  só como chave e não exibe tipo fiscal (CSV: "a classificação fiscal não autoriza uso").
- Card 7: regras PRIVADAS da associação. O Card 13 só trata de regras PÚBLICAS.
- Card 12 (Permits): permits de obra, inspeções e code enforcement. Os processos de zoning
  (audiências, ajustes, variances) são do Card 13, mesmo que também vivam no EnerGov; um
  pedido de "Certificate of Use" ou permit de obra não entra aqui.
- Card 14: usa o zoning só por referência ("see Card 13").
- Conferido em 2026-11-10: nenhum card atual (1, 3, 4, 5) exibe zoning, CDMP ou processo de
  zoning. A página do imóvel lê homeDetails.zoning do MLS (property-detail.mapper.ts:104),
  fora do relatório; o Card 13 NÃO usa o zoning do anúncio (representação comercial; nulo nas
  duas gravações reais).

Proibições invariantes: sem "Allowed", "Compliant", "Buildable", score ou semáforo; o card
nunca afirma que um uso é permitido NO IMÓVEL (só no distrito); parâmetro do distrito nunca
apresentado como aprovação do lote; pavimentos nunca convertidos em pés; -5555/-9999 nunca
viram número nem vazio; sem placeholder em produção; ausência de processo nunca vira "no
zoning issues"; zoning nunca mostrado como se fosse o CDMP (nem o inverso); link externo só do
host oficial.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM EM 2026-11-10) [ESPECÍFICO DO CARD 13]
---------------------------------------------------------------------------------------------
- LegalAnalysisWorkerIds (Application/Contracts/LegalAnalysisContracts.cs:126–130) declara só
  PropertyDataCache e FloodEnvironmentalRisk. Os Cards 6 a 10 existem só como blueprints em
  legal-share; se o Card 13 for o primeiro card novo, ele cria o switch do PromptBuilder
  (seção 5, item 5) e o helper de MLS de 3.3.
- Zoning no codebase: só homeDetails.zoning do MLSDetail, lido por
  RealEstateApiMlsDetailMapper (linhas 111–119) para a página do imóvel; null nas gravações
  reais (RealEstateApiOnlinePropertyDetailTests.cs:633 e :657). Nenhum cliente Regrid nem
  configuração "Regrid" no backend.
- Integrations.Infrastructure/ArcGis/ArcGisQueryClient.cs já faz consultas ArcGIS com limite e
  tratamento de falha: QueryAsync<T> (linha 17), PointParameters (linha 60), Value (78), Date
  (85) e IsFlorida (92); ArcGisSourceOptions guarda URL e limites. É usado pelo
  MiamiDadeEvacuationZoneClient (Card 5), que também normaliza o nome do condado (linhas
  16–20). PointParameters recebe FloodLocationQuery (tipo do Card 5, usado como localização
  genérica); renomear é refatoração separada.
- Camadas de Miami-Dade conferidas em 2026-11-10 (campos lidos do serviço e amostras reais):
  * ZonepolyU_gdb: ZONE, MUNC, OVLY, ZONE_DESC, SHORT_DESC, MODIFIED_DATE;
  * MunicipalZone_gdb: MUNICNAME, ZONE, ZONEDESC, SHORTDESC, MINLOTSIZE (texto), DENSITY
    (texto), FAR, MINLOTWDTH, MAXHEIGHT, GENRLLUTYPE, MAXLOTCOV, MODIFYDATE;
  * TBLUnincorporatedZoneDistrict (tabela): ZONE, MINLOTSIZE, DENSITY, FAR, MINLOTWDTH,
    MAXHEIGHT, GENRLLUTYPE, MIXUSE, MAXLOTCOV (última edição registrada: 2020);
  * Landuse_gdb (CDMP): DESCRIPTION, SOURCE, ACREAGE;
  * Municipalitypoly_gdb: NAME, MUNICID;
  * VLANDMANAGEMENTHEARING_gdb: PROC_NUM, CASE_TYPE, WORK_TYPE, FOLIO, LOC_ADDR, APPL_CITY,
    LAST_RESNO, APPL_LINK, APPL_YEAR, APPL_MONTH, CASE_NUM, APPL_DESC, BOARD, LASTRESNO.
    APPL_DESC traz nome do requerente (ex. "Florida Power and Light Company") e LOC_ADDR o
    endereço: NÃO são lidos.
- Frontend: rota card-flood (app.routes.ts:5–10) como padrão de laboratório; rawPayload é a
  união PropertyDataCacheWorkerPayload | FloodEnvironmentalRiskPayload
  (legal-analysis-result.model.ts:57); guard getFloodEnvironmentalRiskPayload
  (legal-analysis-worker-payload.ts:9–12); composição floodRiskCard em
  pages/listing-detail/components/legal-analysis-result-dialog (.ts:122; bloco @if em
  .html:64; Summary em .html:71). Não existe tabela reutilizável nos cards novos; o Market
  card (shared/market-card) tem tabela própria.
- Imagem de referência do Card 13: AUSENTE em legal-share (recebida na conversa) -> gate 15.7.

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE E APLICABILIDADE [ESPECÍFICO DO CARD 13, SEGUINDO 3.3]
---------------------------------------------------------------------------------------------
- Resolução própria do Card 13, pelos mecanismos obrigatórios de 3.3: PropertyLookup +
  IPropertyDataCacheReader; com FetchMissingSources=true, MLS ausente buscado SOMENTE por
  SharedMlsQueries (DetailAsync/ListingAsync), compartilhando a consulta com os demais workers.
- O registro MLS é só pré-requisito: nada dele é exibido.
- Campos lidos: latitude/longitude (precedência mlsDetail.property -> listing.property ->
  public, como nos Cards 8 e 9), address.stateOrProvince, address.countyOrParish e public.apn
  (folio: só dígitos, 13 em Miami-Dade; inválido -> busca de processos só pelo ponto).
- Coordenadas são obrigatórias para o zoning (camadas e Regrid são consultadas pelo ponto).
  Sem coordenadas válidas -> zoning, CDMP e Regrid unavailable ("coordinatesUnavailable");
  com folio válido, os processos de zoning ainda são buscados por FOLIO.
- Não usar FloodPropertyLocationResolver (é do Card 5 e devolve null sem coordenadas,
  descartando a busca por folio). A validação de coordenadas é própria; unificar é
  refatoração separada.
- Aplicabilidade (pré-requisito, 4.1):
  * FL + Miami-Dade (normalização do condado como no MiamiDadeEvacuationZoneClient) ->
    camadas do condado aplicáveis;
  * outro condado ou estado -> camadas do condado notApplicable; Regrid continua (cobertura
    nacional por condado, conforme o relatório de cobertura dela);
  * condado ausente -> camadas do condado consultadas pelo ponto mesmo assim (o polígono
    decide); ponto fora de todos os polígonos -> coverageMissing.
- Zoning do município ou do condado: decidido pelo polígono de Municipalitypoly_gdb (NAME
  "UNINCORPORATED MIAMI-DADE" -> ZonepolyU_gdb + tabela; outro -> MunicipalZone_gdb).

---------------------------------------------------------------------------------------------
## 9. FONTES DA FASE 1 [ESPECÍFICO DO CARD 13]
---------------------------------------------------------------------------------------------
### 9.1 Cesta MLS — pré-requisito, fonte COMPARTILHADA
- Leitura pela cesta, sem alterar nenhum DTO; busca quando ausente só via SharedMlsQueries
  (3.2). Proibido chamar IOnlinePropertyDetail direto.
### 9.2 Miami-Dade County ArcGIS (conteúdo; só Miami-Dade) — fonte EXCLUSIVA do Card 13
- Serviços hospedados do condado (FeatureServer/0), públicos, sem chave e sem custo. Os mesmos
  dados existem no MapServer gisweb.miamidade.gov/arcgis/rest/services/LandManagement
  (MD_Zoning camadas 1 e 2, MD_CDMP camada 7, MD_ZoningRecords camada 0); usar SÓ os
  hospedados, porque publicam editingInfo.dataLastEditDate (data do rodapé). A escolha é
  confirmada no gate 15.2.
- Consultas (GET .../query, f=json, returnGeometry=false, outFields só com os campos de 6.1):
  1) Municipalitypoly_gdb, ponto (inSR=4326, esriSpatialRelIntersects) -> NAME, MUNICID;
  2) conforme 1): ZonepolyU_gdb ou MunicipalZone_gdb, ponto -> distrito e descrição (e
     parâmetros, no municipal);
  3) não incorporado: TBLUnincorporatedZoneDistrict, where ZONE = '<distrito>' ->
     parâmetros; valor do distrito vindo da própria camada, escapado (sem concatenar texto
     externo sem validação: aceitar só [A-Za-z0-9.-]{1,10});
  4) Landuse_gdb, ponto -> DESCRIPTION;
  5) VLANDMANAGEMENTHEARING_gdb: where FOLIO = '<13 dígitos>' e consulta pelo ponto;
     resultCount até 25;
  6) metadados de cada serviço usado (?f=json) -> editingInfo.dataLastEditDate.
  1, 4, 5 e 6 rodam em paralelo; 2 depende de 1; 3 depende de 2.
- Reaproveitar ArcGisQueryClient.QueryAsync/Value/Date e ArcGisSourceOptions (seção 7).
  Cliente: Integrations.Infrastructure/MiamiDade/Clients/MiamiDadeZoningClient.cs, ao lado
  do cliente de evacuação.
- Mais de um polígono no ponto (borda): distritos iguais -> usa; diferentes -> ambiguous,
  campo "Not available" e limitação "multipleZoningDistricts". Nunca escolher um.
- Estado: found = algum dado; noResults = consulta válida sem polígono/processo;
  unavailable = falha/timeout; ambiguous conforme acima.
### 9.3 Regrid Standardized Zoning (conteúdo; todos os condados cobertos) — EXCLUSIVA
- Documentação lida em 2026-11-10: support.regrid.com/docs/standardized-zoning-schema,
  /docs/standardized-zoning, /docs/getting-started-api, /docs/api-endpoint-overview,
  /docs/additional-endpoints.
- Requisição: GET https://app.regrid.com/api/v2/parcels/point?lat={lat}&lon={lon}&limit=1
  &return_zoning=true&token={token} (limit=1 é o controle de custo indicado pela própria
  Regrid). O zoning vem junto do parcel quando a conta tem o add-on Standardized Zoning; o
  formato exato do bloco de zoning na resposta não está publicado -> gate 15.1.
- Campos do esquema usados: zoning, zoning_description, zoning_type,
  permitted_land_uses_as_of_right, permitted_land_uses_conditional, max_building_height_ft,
  max_density_du_per_acre, min_lot_area_sq_ft, min_front_setback_ft, zoning_data_date,
  municipality_name. Nenhum campo de proprietário do parcel é lido (dono: Card 1).
- Custo: cobrança por registro de parcel devolvido (1 por execução com limit=1; resposta
  vazia não é cobrada); planos self-serve incluem 2.000 registros/mês e excedente de US$ 0,10
  (Standard) ou US$ 0,15 (Premium) por registro; o add-on de zoning é contratado à parte.
  Logar só o fato da chamada; o uso real vem do endpoint de uso da conta.
- Cache: a Regrid permite guardar respostas por até 30 dias. Fase 1 sem cache (1 chamada por
  execução); cache em memória pela coordenada exata só se o gate 15.1 indicar custo
  relevante (decisão 15.4).
- Segredo: o token vai na query string. Mesmo tratamento do AuthKey do Clerk (Card 8) e do
  token da ASCE (Card 9): options/cofre, nunca em log, exceção, payload ou telemetria;
  ConfigureEnvironmentalHttpClient já aplica RemoveAllLoggers.
- Estado: found = parcel com bloco de zoning; noResults = sem parcel no ponto ou parcel sem
  zoning (área não coberta); "NZ" (No Zoning, confirmado pela Regrid com o governo local) ->
  found com distrito "NZ" e descrição "Not zoned"; unavailable = sem token, 401/429/5xx,
  timeout.
### 9.4 Florida Statutes e códigos locais — NÃO exibidos pelo Card 13
- O CSV cita F.S. §163.3194, Miami-Dade Code cap. 33 e Broward Code cap. 39. A imagem não tem
  bloco de base legal: o card não exibe estatutos e não cria entradas no registro único (3.4)
  na fase 1.

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD (worker "permitted-use-zoning", schemaVersion 1) [ESPECÍFICO]
---------------------------------------------------------------------------------------------
{
  schemaVersion, collectedAtUtc,
  jurisdiction: { status, sourceId?, name?, kind?:"municipality"|"unincorporated"|"other",
                  countyCoverage:"miamiDade"|"notIntegrated"|"unconfirmed" },
  district: { status, sourceId?, code?, description? },
  landUsePlan: { status, sourceId?, planName:"CDMP", category? },
  uses: { status, sourceId?, permittedByRight?, conditional? },
  parameters: { status,
                maxHeight?:{ value?, unit:"ft"|"stories", special?:"referToCode"|
                             "notApplicable", sourceId },
                maxDensityUnitsPerAcre?:{ value?, special?, sourceId },
                minLotAreaSqFt?:{ value?, special?, sourceId },
                minFrontSetbackFt?:{ value?, special?, sourceId } },
  zoningRecords: { status, sourceId?, totalCount,
                   items[]:{ process, caseTypeRaw, applicationYear?, applicationMonth?,
                             resolutionReference?, recordUrl? } },
  sourceDataDate?: { value, basis:"oldestContributingSource" },
  sourceStatus[]: formato padrão (4.1),
  limitations[]
}
- sourceId: "miami-dade-municipalities" | "miami-dade-zoning" | "miami-dade-zoning-district-table"
  | "miami-dade-cdmp" | "miami-dade-zoning-hearings" | "regrid-standardized-zoning". status
  das seções: valores de FloodSourceStatus (4.1).
- Status do worker (regra 4.1): fontes de conteúdo = camadas do condado e Regrid. Alguma
  found -> dataAvailable; senão alguma noResults -> noResults; senão -> sourceUnavailable.
  Coordenadas, estado, condado e folio são pré-requisito.
- Limitações (códigos fixos): districtLevelOnly (sempre), coverageAndAccessVary (sempre),
  zoningCodeMismatch, multipleZoningDistricts, heightInStories, municipalRecordsNotIncluded
  (imóvel em município: a camada de processos é do condado), countyNotIntegrated,
  coordinatesUnavailable, folioUnavailable, regridDataDateIsProcessingDate, payloadReduced.
- Sem payload bruto, sem folio, sem endereço, sem nome de requerente (APPL_DESC). Payload alvo
  <= 15 KB (MaxPayloadBytes 60.000); partes pesadas, reduzidas primeiro: zoningRecords.items
  (corte para 10) e textos de usos (corte em 300 caracteres), com "payloadReduced".
- Projeções (classe estática PermittedUseZoningProjection):
  * Read: valida schemaVersion, enums, URLs (só https energov.miamidade.gov) e coleções;
  * ForApi: payload público do card;
  * ForReport: resumo para o LLM — jurisdiction.kind, district.code, district.description,
    landUsePlan.category, parâmetros com unidade, contagem de processos e tipos de caso,
    sourceStatus, limitations. Sem URLs, sem números de processo, sem texto de usos.

---------------------------------------------------------------------------------------------
## 11. BACKEND [ESPECÍFICO DO CARD 13, SEGUINDO 3, 4 E 5]
---------------------------------------------------------------------------------------------
- Application/Contracts/PermittedUseZoningContracts.cs: payload tipado + portas
  IMiamiDadeZoningClient (GetAsync(lat, lon, folio) -> jurisdição, distrito, parâmetros, CDMP,
  processos, datas das camadas e estado por consulta) e IRegridZoningClient (GetAsync(lat,
  lon) -> Found | NoParcel | NoZoning + campos de 9.3; falha por exceção), com DTOs próprios.
- Application/Contracts/PermittedUseZoningProjection.cs: Read, ForApi, ForReport.
- Application/Options/PermittedUseZoningOptions.cs: { SourceTimeout=8 s, TotalBudget=20 s,
  MaxPayloadBytes=60.000, MaxZoningRecords=10 }; validação no
  NewLegalAnalysisModuleDependencyInjection (SourceTimeout <= TotalBudget; TotalBudget <
  WorkerTimeout; MaxPayloadBytes < limite geral; MaxZoningRecords entre 1 e 25), conforme 4.1.
  O orçamento cobre a cadeia de 3 consultas dependentes do condado (9.2).
- Integrations.Infrastructure/MiamiDade/Clients/MiamiDadeZoningClient.cs (reaproveita
  ArcGisQueryClient e ArcGisSourceOptions) e
  Integrations.Infrastructure/Regrid/Clients/RegridZoningClient.cs + options RegridOptions
  { Enabled, BaseUrl, Token } com ValidateOnStart (Enabled exige https no host
  app.regrid.com e Token preenchido). Registro dos dois com ConfigureEnvironmentalHttpClient
  no IntegrationsDependencyInjection, sem BuildRetryPolicy e sem nova tentativa. Sem
  ProjectReference novo (já existe).
- Infrastructure/Workers/PermittedUseZoningWorker.cs:
  Id = LegalAnalysisWorkerIds.PermittedUseZoning ("permitted-use-zoning"); registro explícito
  em AddNewLegalAnalysisWorkers. Sequência: identidade (seção 8) -> condado e Regrid em
  paralelo, cada um isolado com timeout -> regra de precedência e de igualdade de distrito
  (6.1) -> data do rodapé -> limite de payload. Status pela regra 4.1 (seção 10).
  TimeProvider; ILogger + ILegalAnalysisRunTrace com RunId, sourceId, status e durationMs;
  nunca coordenadas, folio, número de processo, token, URL ou payload.
- LegalAnalysisReportPromptBuilder: switch por WorkerId (seção 5, item 5) com
  PermittedUseZoning -> PermittedUseZoningProjection.ForReport; demais inalterados. O LLM
  nunca decide se um uso é permitido no imóvel.
- LegalAnalysisWorkerPayloadMapper: acrescentar PermittedUseZoning ->
  PermittedUseZoningProjection.ForApi, mantendo os casos existentes.
- Não alterar: coordenador, orquestrador, cesta, ArcGisQueryClient (só reutilizado),
  MiamiDadeEvacuationZoneClient, clientes MLS/RentCast, DTOs MLS, persistência, contratos
  HTTP, outros workers.

---------------------------------------------------------------------------------------------
## 12. FRONTEND (Angular) [ESPECÍFICO DO CARD 13, SEGUINDO 4.2 E 5]
---------------------------------------------------------------------------------------------
- data-access/models/legal-analysis-zoning-payload.model.ts (payload) e
  legal-analysis-zoning.model.ts (view model).
- legal-analysis-result.model.ts: PermittedUseZoningPayload na união de rawPayload.
- legal-analysis-worker-payload.ts: getPermittedUseZoningPayload (workerId
  'permitted-use-zoning' + schemaVersion 1), no padrão de getFloodEnvironmentalRiskPayload.
- data-access/legal-analysis-zoning.mapper.ts + .spec + .fixture: payload -> view model
  (title case da jurisdição, sentence case do CDMP, unidades, valores especiais, mês/ano,
  rodapé de fontes e data, segmentos da linha compacta).
- ui/legal-analysis/cards/permitted-use-zoning/: standalone, signals, colapsável (seção 5,
  item 11).
- Layout expandido — a imagem de referência manda (PROMPT, Regra 7):
  1) Cabeçalho: tile com ícone de mapa e pin + divisória vertical + título em gradiente;
     à direita, chip "Illustrative data" (só laboratório, input sampleData) e seta.
  2) Faixa superior em painel claro de largura total, 4 colunas iguais separadas por
     divisórias verticais; cada uma com ícone em tile (prédio público, camadas, casa, folha),
     rótulo com dois-pontos em cima e valor em negrito embaixo.
  3) Linha com dois painéis com borda (esquerdo ~44%, direito ~56%):
     esquerdo: título "District uses" e duas linhas com ícone · rótulo · valor (casa ·
     "Permitted by right:"; pessoas · "Conditional uses:"); título "District parameters" e
     quatro linhas com ícone · rótulo · valor (seta para cima · "Maximum height:"; camadas ·
     "Maximum density:"; quadrado tracejado · "Minimum lot area:"; setas horizontais ·
     "Minimum front setback:"); linhas em faixas alternadas; nota em itálico no rodapé do
     painel;
     direito: título "Zoning applications & resolutions", chip "Illustrative record" (só
     laboratório) e tabela com cabeçalho em faixa (coluna de ícone de documento sem título +
     Process, Case type, Application, Resolution reference, Record); "View source" como
     link sublinhado.
  4) Rodapé abaixo de uma linha divisória: "Sources:" em negrito + fontes à esquerda;
     "Source data date:" em negrito + data à direita ("• Illustrative example" só no
     laboratório); segunda linha em itálico.
- Textos: rótulos exatamente como na imagem (com dois-pontos nos blocos A, B e na faixa);
  ausência "Not available"; unidades "ft", "stories", "units / acre", "sq ft"; números com
  separador de milhar en-US; mês "MMM y".
- Tabela: <table> com <th scope="col">; ícone decorativo com aria-hidden; link "View source"
  com target="_blank", rel="noopener noreferrer" e aria-label "View source for <processo>".
- Linha compacta (está na imagem): segmentos de 6.1, separados por divisórias verticais.
- Responsividade: até 900 px a faixa superior vira grade 2×2 e os dois painéis empilham; a
  tabela ganha rolagem horizontal própria (sem rolagem da página); até 600 px a faixa vira
  1 coluna e cada processo vira um bloco rótulo/valor.
- Rota de teste: features/listings/pages/card-zoning (path "card-zoning", fora do Shell,
  canMatch !environment.production, loadComponent), no padrão de pages/card-flood. Mock com
  os valores da imagem.
- Composição no diálogo: computed zoningCard + bloco @if após o bloco do card de número
  anterior já implementado (Card 12, 11, 10, 9, 8, 7, 6 ou 5) e antes do Summary.
- Regras ESLint de template do AGENTS.md; a11y teclado/leitor de tela. Sem PDF na fase 1.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE
---------------------------------------------------------------------------------------------
Backend (unit): cada campo de 6.1 com fonte primária, alternativa e ausente; regra de
igualdade de distrito (Regrid descartada se divergir); MAXHEIGHT em pavimentos nunca vira pés;
-5555/-9999; "NZ"; não incorporado x município (camada e tabela certas); dois polígonos no
ponto (mesmo distrito e distritos diferentes); validação do código do distrito antes da
consulta à tabela; CDMP fora de Miami-Dade; processos por folio, por ponto e unidos sem
duplicar; ordenação e corte em 10; APPL_LINK fora do host oficial descartado; mês inválido;
data do rodapé = a mais antiga das fontes que contribuíram; sem coordenadas (só processos por
folio); sem folio; fora da FL; Regrid sem token, 401, 429, 5xx, timeout; orçamento/parcial
com TimeProvider fake; isolamento de fonte; limite de payload; projeções Read/ForApi/ForReport;
payload mapper; prompt sem URL, número de processo ou texto de usos.
Backend (segurança): token da Regrid ausente de log, exceção, payload e telemetria; APPL_DESC
e LOC_ADDR nunca lidos.
Backend (reaproveitamento): MLS buscado por SharedMlsQueries (com e sem coordenador no
contexto, como em LegalAnalysisSourceCoordinatorTests); nenhuma chamada direta a
IOnlinePropertyDetail no worker; leitura pela cesta antes de buscar; ArcGisQueryClient
reutilizado; nenhum uso de FloodPropertyLocationResolver; homeDetails.zoning não lido.
Backend (integração): fixtures gravadas das camadas de Miami-Dade (não incorporado e
município) e do gate 15.1 (Regrid com e sem zoning); worker no orquestrador sem afetar
property-data-cache e Flood; WebApiBoundaryTests e suíte completa.
Frontend: specs do mapper (title/sentence case, unidades, especiais, compacto) e do
componente (expandido/compacto/sem processos/fora de Miami-Dade/só Regrid/só condado), guards;
nenhum texto "allowed", "compliant" ou "buildable"; smoke desktop/mobile com capturas no
work item, comparadas com a imagem de referência.
Homologação real: casa não incorporada em Miami-Dade com processo de zoning; imóvel em
município de Miami-Dade; imóvel da FL fora de Miami-Dade (só Regrid); Regrid sem token; ponto
sem cobertura. Conferir no log/LegalAnalysis.txt os eventos Worker e SourceQuery.
Aceite: todos os campos da imagem dinâmicos e com fonte; cards já implementados e Summary
inalterados; nenhum dado de outro card repetido; nenhum valor simulado; nenhuma ausência
tratada como "permitido" ou "sem processo"; execução total < 120 s.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO
---------------------------------------------------------------------------------------------
1) Validação de dados e fontes: este documento + conferência anti-duplicidade + decisões
   15.3 e 15.6 + gate 15.1 (contratação do add-on de zoning da Regrid: iniciar já).
2) Rota card-zoning com HTML estático nas versões expandida e compacta, fiel à imagem;
   aprovação visual.
3) Worker com as camadas de Miami-Dade (jurisdição, distrito, descrição, parâmetros do
   condado, CDMP, processos): options, registro no DI, cliente, projeções, PromptBuilder,
   PayloadMapper, modelo/guard/mapper/componente Angular, composição no diálogo. Usos e recuo
   saem "Not available" com sourceStatus da Regrid "notConfigured".
4) Regrid como incremento aditivo, quando o 15.1 estiver confirmado: preenche usos, recuo e
   altura em pés no mesmo payload, e é a fonte fora de Miami-Dade; o card não muda de forma.
5) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile, execução real e leitura do log.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
---------------------------------------------------------------------------------------------
15.1 Regrid: contratar API + add-on Standardized Zoning; confirmar custo por registro, cota e
     limite de taxa; confirmar a cobertura de zoning de Miami-Dade e Broward no relatório de
     cobertura; gravar UMA resposta real de /api/v2/parcels/point (limit=1, return_zoning)
     para um imóvel de Miami-Dade e anotar: onde vem o bloco de zoning, formato real de
     permitted_land_uses_as_of_right/_conditional, tipos dos números e de
     zoning_data_date. Confirmar nos termos a exibição no relatório e o armazenamento no
     pacote de auditoria. Bloqueia o passo 4.
15.2 Miami-Dade: confirmar com o condado (ou na documentação da camada) a unidade de MAXHEIGHT
     (amostra RU-1 = 2 -> pavimentos), a unidade de DENSITY (unidades por acre) e de
     MINLOTSIZE (pés²); a atualidade de TBLUnincorporatedZoneDistrict (última edição em 2020)
     frente ao Code cap. 33 vigente; e o uso dos serviços hospedados no lugar do MapServer.
15.3 CDMP dentro de municípios: a imagem tem um único rótulo "CDMP land-use category".
     Recomendação: exibir o CDMP do condado em todo Miami-Dade (é o que a camada publica) e,
     em município, acrescentar a limitação de que o plano municipal também rege; alternativa:
     trocar o valor por MunicipalFutureLandUse_gdb.FLUDESC com o rótulo "Municipal land-use
     category", o que altera a imagem e exige aprovação visual.
15.4 Cache da Regrid (permitido por até 30 dias): decidir após o 15.1, conforme custo.
15.5 Exibição dos números de processo e do link EnerGov: são registros públicos do condado;
     confirmar que podem ir ao pacote de auditoria do Blob. APPL_DESC (nome do requerente) não
     é lido por desenho.
15.6 Escopo: confirmar que a fase 1 é só o que a imagem mostra e que "Restrições de locação
     pública" (CSV; PROMPT 13B) vai para a fase 2. Atenção: o blueprint do Card 7 manda
     exibir "Public rental rules: see Card 13"; enquanto o Card 13 não tiver esse bloco, o
     Card 7 deve omitir o indicador ou apontar para a fase 2 (ajustar no Card 7).
15.7 Imagem de referência: salvar em legal-share (padrão CARD13_ZONING_IMG_REF.<ext>).
15.8 Uso de LLM no card: em aberto (arquiteto Bruno, PROMPT 13E). O worker entrega fatos; o
     LLM só recebe ForReport.
15.9 Base de código (reconferido em 2026-11-10): d_leevli-platform-api está em
     feat/legal-analysis-source-coordinator (57074dd6 "Modulariza Workers"), que contém o
     Card 5 e o Plano A; d_leevli-platform-web está em LEGAL (71e79913). Os Cards 6 a 12 não
     estão implementados: as peças compartilhadas (switch do PromptBuilder, helper de MLS de
     3.3) são criadas por quem for implementado primeiro; os outros só acrescentam.
15.10 Work item com critérios de aceite; branch feat/<card>-permitted-use-zoning; commits
     feat(new-legal-analysis): #<card> ...

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
