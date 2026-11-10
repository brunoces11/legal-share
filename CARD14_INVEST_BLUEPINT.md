# LEEVLI | LEGAL ANALYSIS | CARD 14 — RENTAL AND INVESTMENT
# MASTER IMPLEMENTATION BLUEPRINT V1 — BASEADO NO TEMPLATE DO CARD 6 (CARD6_HOA_FINANCIAL.md, V3)
# (aderente ao código em 2026-11-10, incluindo o Plano A: coordenador de consultas por execução)

Este documento é o blueprint de implementação do Card 14 (Rental and Investment).
Segue a estrutura do template do Card 6: as seções [REGRA FIXA] são idênticas às do template;
as seções [ESPECÍFICO DO CARD] são próprias do Card 14.

V1 (2026-11-10): a IMAGEM DE REFERÊNCIA do card é o contrato visual. Ela traz a versão
expandida e a compacta. Todo dado exibido está mapeado, na seção 6.1, para um campo real de
fonte programática JÁ INTEGRADA no codebase: RentCast (estimativa de aluguel de longo prazo
e comparáveis de aluguel, documentação lida em 2026-11-10) e a cesta MLS da RealEstateAPI
(preço pedido). As seções [REGRA FIXA] foram reconferidas contra o código em 2026-11-10: as
referências de arquivo e linha datadas de 2026-11-06 continuam válidas.

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
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 14]
---------------------------------------------------------------------------------------------
Card do POTENCIAL DE ALUGUEL do imóvel para quem compra para alugar: estimativa de aluguel de
longo prazo, preço pedido, a relação bruta aluguel mensal / preço pedido e os comparáveis de
aluguel que sustentam a estimativa. Todo dado vem de fonte programática e carrega a origem;
o card não calcula retorno, fluxo de caixa, cap rate nem recomenda a compra.

Termos do card:
- Aluguel estimado ...... estimativa da RentCast para locação de longo prazo do imóvel
                          ("rent"); para Multi-Family/Apartment vale por UNIDADE, não pelo
                          prédio (documentação da RentCast).
- Preço pedido .......... preço do anúncio MLS (asking price). Não é avaliação nem preço de
                          venda.
- Relação bruta mensal .. aluguel estimado / preço pedido x 100, antes de qualquer custo
                          ("Gross monthly ratio before costs", texto da imagem).
- Comparável de aluguel . anúncio de aluguel próximo que a RentCast usou na estimativa, com o
                          aluguel PEDIDO (não é contrato assinado).

### 6.1 CONTRATO VISUAL — IMAGEM DE REFERÊNCIA -> DADOS
Imagem recebida em 2026-11-10, com o estado expandido e o compacto. São marcas do
laboratório, não dados (seção 12): o chip "Illustrative data", o chip "Illustrative
listings", o trecho "and for demonstration only" da nota da tabela e o texto "Values shown
are illustrative. This is not live data."
Formato de cada linha: "campo (exemplo da imagem) <- fonte | estado quando ausente".

Cabeçalho
- Ícone de casa + título "Rental & Investment" (grafia da imagem, com "&") + seta. Sem dado.

Faixa de indicadores (3 blocos com ícone, separados por divisórias verticais)
- Estimated monthly rent ($3,000 / month)
    <- RentCast GET /v1/avm/rent/long-term: `rent` — JÁ consultado em toda execução pelo
       property-data-cache (rentCastRent) e em cache por 24 h no RentCastClient. Inteiro em
       dólares, "$<valor> / month".
    |  "Unavailable"
- Asking price ($500,000)
    <- cesta MLS, MESMA precedência do Card 1 (legal-analysis-property-card.mapper.ts:121):
       mlsDetail.listPrice -> mlsSearchDetail.price -> mlsSearchDetail.listing.listPriceLow.
       Os dois cards nunca podem mostrar valores diferentes (decisão 15.2).
    |  "Unavailable"
- Monthly rent / asking price (0.60%)
    <- cálculo determinístico no backend: rent / askingPrice x 100, arredondado em 2 casas
       ("0.60%"), só com os dois valores > 0.
    |  "Unavailable" quando falta qualquer um dos dois.
- Linha abaixo da faixa (texto fixo da imagem): "Estimated long-term rent • Gross monthly
  ratio before costs".

Bloco "Rental comparables"
- Tabela Property · Beds · Baths · Sq ft · Asking rent / month · Distance, uma linha por
  comparável, com ícone de casa. Fonte: `comparables[]` da MESMA resposta da RentCast
  (/v1/avm/rent/long-term), na ordem da RentCast (correlation decrescente: "most similar
  listings appearing first"). Exibidos: os 3 primeiros válidos (a imagem mostra 3;
  MaxDisplayedComparables).
  * Property (101 Example Ave) <- addressLine1; sem ele, o trecho de formattedAddress antes
    da primeira vírgula. Sem os dois -> comparável descartado.
  * Beds (3) <- bedrooms (0 = studio -> "Studio").
  * Baths (2) <- bathrooms (aceita fração: "2.5").
  * Sq ft (1,600) <- squareFootage (separador de milhar).
  * Asking rent / month ($2,950) <- price ("The listed price or rent of the property
    listing"). Sem price > 0 -> comparável descartado.
  * Distance (0.4 mi) <- distance ("in miles"), 1 casa decimal.
  * Campo ausente em comparável válido -> "—".
- Nota abaixo da tabela: na imagem, "These rental comparable fields are documented by
  RentCast and for demonstration only." No card definitivo (proposta para aprovar, 15.7):
  "Rental comparables from RentCast. Asking rents from rental listings, not signed leases."
- Sem comparáveis válidos: o bloco mostra "No rental comparables returned" (não está na
  imagem; proposta, 15.7). A estimativa continua exibida.

Rodapé — "Source: RentCast estimate & rental comparables • Listing asking price"
- Montado a partir de sourceStatus: só as fontes que responderam com dado. Texto fixo é
  proibido.

Versão compacta (está na imagem)
- "Rental & Investment | $<rent> / month estimated rent | $<preço> asking price |
  <ratio>% monthly gross" + seta para baixo; segmento ausente -> "Unavailable" no lugar do
  valor.

Regra de leitura do contrato: nenhum valor é fixo no card definitivo; são fixos só rótulos,
a linha de definição abaixo da faixa e a nota da tabela.

### 6.2 Fases
Fase 1 (este blueprint, 1 PR/1 card) = exatamente a imagem:
- RentCast rent estimate (já integrada e já consultada por execução) + seus comparáveis
  (hoje descartados pelo DTO — 7 e 9.2).
- Cesta MLS: preço pedido.
- Cálculo da relação bruta mensal no backend.
Fase 2 (backlog, card/PR próprio e nova imagem), itens do PROMPT 14B que a imagem NÃO mostra
— decisão 15.6:
- Custos conhecidos: taxa da associação (dono Card 6), impostos (dono Card 11) — só por
  referência; cenários de receita e retorno com hipóteses explícitas.
- Restrições de locação: públicas (dono Card 13) e privadas da associação (dono Card 7) — só
  por referência ("see Card 13" / "see Card 7").
- "Aprovação FHA do condomínio" (CSV: HUD FHA Condo Search; acesso programático não
  confirmado).
- Estatísticas de aluguel do ZIP (regionalMarket.rent da RentCast, já na cesta e hoje só na
  seção técnica do diálogo).
- Short-term rental: fora de escopo (a RentCast usada é long-term).

### 6.3 Fronteira com outros cards (regra anti-duplicidade, ⚠️ item 4)
- Card 1 (Property Identity) já EXIBE "Asking price" (legal-analysis-property-card.mapper.ts:
  121 e 126). A imagem do Card 14 também o exibe, como base da relação aluguel/preço.
  CONFLITO com o ⚠️ item 4 -> decisão 15.2 (mesmo caso do "Year built" no Card 9).
  Recomendação: seguir a imagem como exceção documentada, com a mesma fonte e a mesma
  precedência do Card 1. Alternativa: "Asking price: see Card 1", mantendo o cálculo.
- Estimativa de aluguel: nenhum CARD do relatório a exibe; aparece só na seção técnica do
  diálogo ("Rental estimate", legal-analysis-result-dialog.ts:190–197), que é detalhamento
  técnico, não card. O Card 14 é o DONO.
- Card 1 pode exibir "Estimated price" (rentCastAvm, opcional). O Card 14 não usa a AVM de
  venda.
- Card 3 (Recent Sales) e Card 4 (Active Listings) mostram comparáveis de VENDA. Os
  comparáveis do Card 14 são de ALUGUEL (outra resposta, outro dado). Sem repetição.
- Cards 6, 7, 11 e 13: custos e restrições, só referenciados na fase 2.
- Conferido em 2026-11-10: nenhum card atual (1, 3, 4, 5) exibe aluguel estimado,
  comparáveis de aluguel ou relação aluguel/preço.

Proibições invariantes: sem "Good investment", "High yield", nota, score, semáforo, ROI, cap
rate, cash flow, retorno líquido ou "1% rule passed"; a relação é sempre rotulada "gross,
before costs"; estimativa nunca apresentada como aluguel garantido ou contratado; aluguel de
comparável é sempre "asking rent"; para Multi-Family/Apartment a estimativa é rotulada por
unidade; preço pedido nunca apresentado como valor de mercado; null nunca vira 0; sem
placeholder em produção.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM EM 2026-11-10) [ESPECÍFICO DO CARD 14]
---------------------------------------------------------------------------------------------
- LegalAnalysisWorkerIds (Application/Contracts/LegalAnalysisContracts.cs:126–130) declara só
  PropertyDataCache e FloodEnvironmentalRisk. Os Cards 6 a 10 e 13 NÃO estão implementados
  (só blueprints em legal-share).
- RentCast JÁ integrada:
  * IRentCastClient (Properties.Application/Interfaces/IRentCastClient.cs:3–8):
    GetAvmAsync, GetRentEstimateAsync, GetMarketStatsByZipAsync.
  * RentCastRentResult (linha 37) = (Rent, RentRangeLow, RentRangeHigh), SEM comparáveis.
    RentCastAvmResult (linhas 10–13) já tem Comparables (RentCastSaleComparable, linhas
    15–35: formattedAddress, addressLine1, bedrooms, bathrooms, squareFootage, price,
    distance, correlation...).
  * RentCastRentResponseDto (Integrations.Infrastructure/RentCast/Dtos) declara só Rent,
    RentRangeLow e RentRangeHigh: os `comparables` que a API devolve são DESCARTADOS na
    desserialização.
  * RentCastClient.GetRentEstimateAsync (Clients/RentCastClient.cs:93–125): GET
    avm/rent/long-term?address=..., cache-aside em IMemoryCache por 24 h (CacheLifetime,
    linha 17), timeout de 5 s (CallTimeout, linha 18). Devolve null tanto sem estimativa
    quanto em falha (SendAsync, linhas 264–360). O cliente já tem um padrão de resultado que
    separa desfechos: NeighborhoodMarketLookup (linhas 224–227 e 257–258).
  * RentCastBudgetHandler: teto diário da plataforma que conta TODA tentativa HTTP à RentCast.
- PropertyDataCacheWorker (NewLegalAnalysis.Infrastructure/Workers):
  * deriva o endereço do formato RentCast a partir do registro MLS quando o snapshot não o
    traz (linhas 38–78) e usa PropertyLookup.RentCastAddress;
  * busca rentCastRent DIRETO no cliente, sem coordenador (linhas 106–107);
  * publica rentCastRent no payload do property-data-cache (linha 130), que o
    LegalAnalysisReportPromptBuilder envia ao LLM sem projeção.
- Consumidores de RentCastRentResult: PropertyDataCacheResult (Properties.Application/DTOs,
  linha 17), PropertyDataCacheWorker, GetOnlineLeevliScoreHandler (linhas 56 e 74),
  LeevliScoreCalculator (linha 160) e os testes PropertyDataCacheReaderTests,
  PropertyDataCacheWorkerTests, GetOnlineLeevliScoreHandlerTests, LeevliScoreCalculatorTests.
- Preço pedido: o Card 1 lê detail.listPrice -> search.price -> listing.listPriceLow
  (legal-analysis-property-card.mapper.ts:121) e o exibe como "Asking price" (linha 126).
- Frontend: rentCastRent só aparece na seção técnica do diálogo
  (legal-analysis-result-dialog.ts:190–197). Rota card-flood (app.routes.ts:5–10) como padrão
  de laboratório; rawPayload é a união PropertyDataCacheWorkerPayload |
  FloodEnvironmentalRiskPayload (legal-analysis-result.model.ts:57); guard
  getFloodEnvironmentalRiskPayload (legal-analysis-worker-payload.ts:9–12); composição
  floodRiskCard (dialog .ts:122; bloco @if em .html:64; Summary em .html:71).
- LegalAnalysisSourceKeys declara só MlsDetail e MlsListing.
- CSV: a RentCast NÃO aparece no Fontes_API_Legal_analyses.csv. Os itens do CSV ligados ao
  Card 14 são "Dados de anúncio e MLS" (preço pedido), "Restrições de locação pública",
  "Regras de aluguel", "Taxa mensal da associação" e "Aprovação FHA do condomínio"; o PROMPT
  (14D) registra que o CSV não confirma fonte de aluguel comparável. A RentCast já é fonte
  integrada e usada pelo relatório (property-data-cache) — divergência registrada em 15.8.
- Imagem de referência do Card 14: AUSENTE em legal-share (recebida na conversa) -> 15.7.

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE E APLICABILIDADE [ESPECÍFICO DO CARD 14, SEGUINDO 3.3]
---------------------------------------------------------------------------------------------
- Resolução própria do Card 14, pelos mecanismos obrigatórios de 3.3: PropertyLookup +
  IPropertyDataCacheReader; com FetchMissingSources=true, MLS ausente buscado SOMENTE por
  SharedMlsQueries (DetailAsync/ListingAsync), compartilhando a consulta com os demais workers.
- No Card 14 o registro MLS é fonte de conteúdo (preço pedido) e pré-requisito (endereço para
  a RentCast, tipo do imóvel).
- Endereço RentCast: EXATAMENTE o mesmo texto que o property-data-cache usa (mesmos campos do
  registro MLS — unparsedAddress, city, stateOrProvince, zipCode — passados por
  PropertyLookup.From / PropertyDataAddressFormatter.ForRentCast). Motivo: a chave do cache
  da RentCast e a chave do coordenador dependem desse texto; um formato diferente faria uma
  segunda chamada. A derivação hoje está dentro do PropertyDataCacheWorker (linhas 38–78);
  o Card 14 a reproduz pelo helper de 3.3 sem alterar aquele worker.
- Campos lidos: listPrice / price / listPriceLow (precedência de 6.1), endereço (acima) e
  propertyType/propertySubType (só para a limitação de multi-family). Nada além disso.
- Não usar FloodPropertyLocationResolver (3.3); o card não precisa de coordenadas.
- Aplicabilidade (pré-requisito, 4.1):
  * endereço RentCast montado -> consulta;
  * sem endereço -> RentCast não consultada (unavailable, "addressUnavailable"); o preço
    pedido continua;
  * a RentCast cobre os EUA; resposta sem `rent` -> noResults.

---------------------------------------------------------------------------------------------
## 9. FONTES DA FASE 1 [ESPECÍFICO DO CARD 14]
---------------------------------------------------------------------------------------------
### 9.1 Cesta MLS (RealEstateAPI) — conteúdo (preço pedido) e pré-requisito, fonte COMPARTILHADA
- Leitura pela cesta, sem alterar nenhum DTO; busca quando ausente só via SharedMlsQueries
  (3.2). Proibido chamar IOnlinePropertyDetail direto. Nenhuma chamada nova à RealEstateAPI.
- Preço pedido: decimal > 0; limitação fixa "askingPriceAsListed" (CSV "Dados de anúncio e
  MLS": informação comercial que exige confirmação documental).
### 9.2 RentCast — Rent Estimate (long-term) — conteúdo, fonte COMPARTILHADA (property-data-cache e Card 14)
- Documentação lida em 2026-11-10: developers.rentcast.io/reference/rent-estimate-long-term e
  /reference/property-valuation-schema.
- Requisição já usada pelo codebase: GET https://api.rentcast.io/v1/avm/rent/long-term
  ?address=... (header X-Api-Key). Parâmetros opcionais (compCount 5–25, padrão 15;
  maxRadius; daysOld; lookupSubjectAttributes, padrão true) NÃO são alterados: mudariam a
  resposta compartilhada com o property-data-cache e com o Leevli Score.
- Resposta (documentada): rent, rentRangeLow, rentRangeHigh, subjectProperty{...},
  comparables[]{ id, formattedAddress, addressLine1, addressLine2, city, state, zipCode,
  latitude, longitude, propertyType, bedrooms, bathrooms, squareFootage, lotSize, yearBuilt,
  status, price, listingType, listedDate, removedDate, lastSeenDate, daysOnMarket, distance,
  daysOld, correlation }. Schema: price = "The listed price or rent of the property
  listing"; distance = "in miles"; comparables ordenados por correlation decrescente.
- Mudança ADITIVA necessária (afeta Properties/Integrations — rodar os testes dos dois
  módulos e do NewLegalAnalysis):
  1) RentCastRentResponseDto ganha Comparables (lista nullable, reutilizando
     RentCastSaleComparable, que já tem todos os campos de 6.1 — como a AVM já faz);
  2) RentCastRentResult ganha Comparables marcada com [JsonIgnore]. Motivo: o
     IPropertyDataCacheReader serializa esse objeto para o payload do property-data-cache,
     que vai ao LLM e ao Blob; com [JsonIgnore] esse payload, os Cards 1/3/4 e o prompt
     continuam byte a byte iguais. O IMemoryCache guarda o objeto, não o JSON, então os
     comparáveis ficam disponíveis no cache do cliente;
  3) GetOnlineLeevliScoreHandler e LeevliScoreCalculator não mudam.
- Consulta compartilhada (regra 3.2 — a fonte é usada por 2 workers na mesma execução):
  * Hoje o property-data-cache chama o cliente DIRETO (linhas 106–107). Se o Card 14 também
    chamar, as duas chamadas podem sair ao mesmo tempo (ambas com cache vazio) e o
    RentCastBudgetHandler conta as duas.
  * Solução, no padrão de SharedMlsQueries: helper interno SharedRentCastQueries
    .RentEstimateAsync(context, address, waitToken) -> context.QuerySourceAsync(
    LegalAnalysisSourceKeys.RentCastRent(address), ...). O coordenador exige o MESMO tipo de
    resultado para a mesma chave, e o contrato precisa separar falha de "sem estimativa"
    (3.2): acrescentar ao IRentCastClient um método que devolve desfecho + resultado (no
    padrão do NeighborhoodMarketLookup já existente: Found | NoEstimate | Unavailable |
    BudgetExhausted), e manter GetRentEstimateAsync como está para os consumidores atuais.
  * O property-data-cache passa a chamar o MESMO helper na linha 107 — mudança de uma linha
    em outro worker, sem alterar seu payload nem sua semântica. Exige autorização do
    arquiteto (⚠️ item 6) -> decisão 15.3.
  * Depois da primeira resposta, o cache de 24 h do cliente atende os demais pedidos sem
    nova chamada.
- Leitura do Card 14: 1) cesta (rentCastRent presente = há resultado no cache do cliente);
  2) obter o resultado tipado com comparáveis pelo helper (com cache presente não há
  chamada HTTP); 3) sem cache e FetchMissingSources=true -> o helper consulta; com
  FetchMissingSources=false (prévia, somente leitura) e sem cache -> unavailable
  "notInCache", sem chamada.
- Estado da fonte: found = rent presente; noResults = resposta válida sem rent;
  unavailable = falha, timeout, sem endereço ou fora do cache na prévia; budgetExhausted ->
  unavailable "budgetExhausted".
- Comparáveis: até MaxDisplayedComparables (3) válidos (6.1), na ordem da RentCast. Sem
  coordenadas, sem addressLine2 e sem datas no payload.

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD (worker "rental-investment", schemaVersion 1) [ESPECÍFICO]
---------------------------------------------------------------------------------------------
{
  schemaVersion, collectedAtUtc,
  rentEstimate: { status, sourceId:"rentcast-rent-long-term", term:"longTerm",
                  amount?:{ value, currency:"USD" }, rangeLow?, rangeHigh?,
                  perUnit: true|false },
  askingPrice: { status, sourceId:"realestateapi-mls",
                 amount?:{ value, currency:"USD" },
                 sourcePath?:"mlsDetail.listPrice"|"mlsSearchDetail.price"|
                             "mlsSearchDetail.listing.listPriceLow" },
  grossMonthlyRatio?: { percent, basis:"rentEstimateOverAskingPrice", beforeCosts:true },
  rentalComparables: { status, sourceId:"rentcast-rent-long-term", totalReturned,
                       items[]:{ address, bedrooms?, bathrooms?, squareFootage?,
                                 askingRent, distanceMiles? } },
  sourceStatus[]: formato padrão (4.1),
  limitations[]
}
- status das seções: valores de FloodSourceStatus (4.1).
- rangeLow/rangeHigh vão no payload (contexto de incerteza para o LLM) e NÃO são exibidos
  (a imagem não os mostra).
- percent com 2 casas, calculado no backend; o LLM e o frontend nunca recalculam.
- Status do worker (regra 4.1): fontes de conteúdo = RentCast e cesta MLS (preço pedido).
  Alguma found -> dataAvailable; senão alguma noResults -> noResults; senão ->
  sourceUnavailable. O endereço é pré-requisito e não gera dataAvailable.
- Limitações (códigos fixos): estimateNotGuaranteed (sempre que houver estimativa),
  askingPriceAsListed, comparablesAreAskingRents (sempre que houver comparável),
  perUnitEstimate (Multi-Family/Apartment), ratioBeforeCosts (sempre que houver relação),
  addressUnavailable, notInCache, budgetExhausted, payloadReduced.
- Sem payload bruto da RentCast, sem coordenadas, sem endereço do imóvel avaliado. Payload
  alvo <= 3 KB (MaxPayloadBytes 20.000).
- Projeções (classe estática RentalInvestmentProjection):
  * Read: valida schemaVersion, enums, valores > 0 e coleções antes de qualquer uso;
  * ForApi: payload público do card;
  * ForReport: resumo para o LLM — rent, faixa, perUnit, askingPrice, percent, quantidade
    de comparáveis e mediana do aluguel pedido deles, sourceStatus, limitations. Sem
    endereços dos comparáveis.

---------------------------------------------------------------------------------------------
## 11. BACKEND [ESPECÍFICO DO CARD 14, SEGUINDO 3, 4 E 5]
---------------------------------------------------------------------------------------------
- Application/Contracts/RentalInvestmentContracts.cs: payload tipado (sem porta nova: usa
  IRentCastClient).
- Application/Contracts/RentalInvestmentProjection.cs: Read, ForApi, ForReport.
- Application/Options/RentalInvestmentOptions.cs: { SourceTimeout=8 s, TotalBudget=12 s,
  MaxPayloadBytes=20.000, MaxDisplayedComparables=3 }; validação no
  NewLegalAnalysisModuleDependencyInjection (SourceTimeout <= TotalBudget; TotalBudget <
  WorkerTimeout; MaxPayloadBytes < limite geral; MaxDisplayedComparables entre 1 e 10),
  conforme 4.1. A RentCast já tem timeout próprio de 5 s no cliente.
- Properties.Application/Interfaces/IRentCastClient.cs: Comparables em RentCastRentResult
  ([JsonIgnore]) + método com desfecho (9.2). Integrations.Infrastructure/RentCast:
  Comparables no RentCastRentResponseDto e o método novo no RentCastClient, reaproveitando
  SendAsync, cache e budget existentes (mesma chave de cache "rent" + endereço).
- Infrastructure/Orchestration/LegalAnalysisSourceKeys.cs: RentCastRentLabel e
  RentCastRent(address).
- Infrastructure/Workers/SharedRentCastQueries.cs: helper no padrão de SharedMlsQueries.
- Infrastructure/Workers/RentalInvestmentWorker.cs:
  Id = LegalAnalysisWorkerIds.RentalInvestment ("rental-investment"); registro explícito em
  AddNewLegalAnalysisWorkers. Sequência: identidade (seção 8) -> preço pedido (cesta) ->
  estimativa e comparáveis (9.2, isolada com timeout) -> relação bruta -> comparáveis
  válidos -> payload. Status pela regra 4.1 (seção 10). TimeProvider; ILogger +
  ILegalAnalysisRunTrace com RunId, sourceId, status e durationMs; nunca endereço, valores,
  chave ou payload.
- PropertyDataCacheWorker: SOMENTE a linha 107 passa a usar SharedRentCastQueries (15.3).
- LegalAnalysisReportPromptBuilder: switch por WorkerId (seção 5, item 5) com
  RentalInvestment -> RentalInvestmentProjection.ForReport; demais inalterados. O LLM nunca
  calcula relação, retorno ou recomendação.
- LegalAnalysisWorkerPayloadMapper: acrescentar RentalInvestment ->
  RentalInvestmentProjection.ForApi, mantendo os casos existentes.
- Não alterar: coordenador, orquestrador, cesta (formato do payload do property-data-cache),
  parâmetros da chamada RentCast, chave e validade do cache RentCast, budget, Leevli Score,
  persistência, contratos HTTP, outros workers (exceto a linha de 15.3).

---------------------------------------------------------------------------------------------
## 12. FRONTEND (Angular) [ESPECÍFICO DO CARD 14, SEGUINDO 4.2 E 5]
---------------------------------------------------------------------------------------------
- data-access/models/legal-analysis-rental-payload.model.ts (payload) e
  legal-analysis-rental.model.ts (view model).
- legal-analysis-result.model.ts: RentalInvestmentPayload na união de rawPayload.
- legal-analysis-worker-payload.ts: getRentalInvestmentPayload (workerId 'rental-investment'
  + schemaVersion 1), no padrão de getFloodEnvironmentalRiskPayload.
- data-access/legal-analysis-rental.mapper.ts + .spec + .fixture: payload -> view model
  (moeda en-US sem centavos, "/ month", percentual com 2 casas, banheiros com fração,
  "Studio", distância "<n> mi", rodapé dinâmico, linha compacta).
- ui/legal-analysis/cards/rental-investment/: standalone, signals, colapsável (seção 5,
  item 11).
- Layout expandido — a imagem de referência manda (PROMPT, Regra 7):
  1) Cabeçalho: tile com ícone de casa + divisória vertical + título em gradiente + chip
     "Illustrative data" (só laboratório, input sampleData) + seta.
  2) Faixa de indicadores em painel claro: três blocos de mesma largura, cada um com tile de
     ícone (moedas, etiqueta, percentual), rótulo e valor em negrito; divisórias verticais.
  3) Linha em itálico: "Estimated long-term rent • Gross monthly ratio before costs".
  4) Painel "Rental comparables" (título em negrito; chip "Illustrative listings" só no
     laboratório) com tabela de cabeçalho em faixa clara: Property (ícone de casa +
     endereço), Beds, Baths, Sq ft, Asking rent / month, Distance; números centralizados
     como na imagem; nota em itálico abaixo.
  5) Divisória + rodapé "Source: ..." à esquerda. O texto "Values shown are illustrative.
     This is not live data." à direita existe só no laboratório.
- Linha compacta (está na imagem): tile de ícone + título + três segmentos com ícone e
  divisórias ("$<rent> / month estimated rent", "$<preço> asking price", "<ratio>% monthly
  gross") + seta.
- Tabela acessível: <table> com <caption> visualmente oculto ("Rental comparables"),
  <th scope="col">; ícones decorativos com aria-hidden.
- Responsividade: até 900 px a faixa de indicadores vira 3 linhas empilhadas; até 760 px a
  tabela vira lista de cartões (endereço em destaque, demais campos em pares rótulo · valor);
  na linha compacta, até 600 px os segmentos quebram em duas linhas.
- Rota de teste: features/listings/pages/card-invest (path "card-invest", fora do Shell,
  canMatch !environment.production, loadComponent), no padrão de pages/card-flood, com mocks
  dos estados: completo, sem preço pedido, sem estimativa, sem comparáveis.
- Composição no diálogo: computed rentalInvestmentCard + bloco @if após o bloco do card de
  número anterior já implementado (Card 13, ..., 5) e antes do Summary.
- Regras ESLint de template do AGENTS.md (table-scope, elements-content). Sem PDF na fase 1.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE
---------------------------------------------------------------------------------------------
Backend (unit): rent presente/ausente; faixa; Multi-Family/Apartment -> perUnit; preço pedido
pela precedência do Card 1 em todas as combinações; relação com os dois valores, sem um deles
e arredondamento (3000/500000 -> 0.60); comparáveis válidos, sem endereço, sem price,
com bedrooms 0, bathrooms fracionário, distance ausente; limite de 3 na ordem da RentCast;
desfechos Found/NoEstimate/Unavailable/BudgetExhausted; prévia sem cache não chama a API;
orçamento com TimeProvider fake; isolamento da fonte; limite de payload; projeções
Read/ForApi/ForReport; payload mapper; prompt sem endereços de comparáveis.
Backend (compartilhamento e não regressão): property-data-cache e Card 14 na mesma execução
fazem UMA chamada à RentCast rent (chave idêntica; "Joined" no log); payload do
property-data-cache idêntico ao atual (Comparables com [JsonIgnore]); PropertyDataCacheReader
Tests, PropertyDataCacheWorkerTests, GetOnlineLeevliScoreHandlerTests e
LeevliScoreCalculatorTests inalterados e verdes; GetRentEstimateAsync com o mesmo
comportamento.
Backend (reaproveitamento): MLS buscado por SharedMlsQueries (com e sem coordenador no
contexto, como em LegalAnalysisSourceCoordinatorTests); RentCast só por SharedRentCastQueries;
nenhuma chamada direta a IOnlinePropertyDetail ou ao RentCastClient no worker; endereço
RentCast idêntico ao do property-data-cache.
Backend (integração): fixture gravada de /v1/avm/rent/long-term com comparables (gate 15.1);
worker no orquestrador sem afetar property-data-cache e Flood; WebApiBoundaryTests e suíte
completa.
Frontend: specs do mapper (formatos, estados ausentes, rodapé dinâmico, linha compacta), do
componente (expandido/compacto/tabela responsiva) e dos guards; mesmo "Asking price" do
Card 1 para o mesmo payload de cesta; nenhum texto "yield", "ROI", "cap rate" ou "good
investment" renderizado; smoke desktop/mobile com capturas no work item, comparadas com a
imagem de referência.
Homologação real: casa unifamiliar ativa na FL; condomínio; imóvel Multi-Family; anúncio sem
preço; RentCast sem estimativa; RentCast com budget esgotado. Conferir no
log/LegalAnalysis.txt os eventos Worker e SourceQuery (Started/Joined) da RentCast.
Aceite: todos os campos da imagem dinâmicos e com fonte; o mesmo preço pedido do Card 1;
payload do property-data-cache inalterado; cards já implementados e Summary inalterados;
nenhum dado de outro card repetido além da exceção aprovada em 15.2; uma chamada RentCast
rent por execução; execução total < 120 s.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO
---------------------------------------------------------------------------------------------
1) Validação de dados e fontes: este documento + conferência anti-duplicidade + decisões
   15.2, 15.3 e 15.6 + gate 15.1.
2) Rota card-invest com HTML estático nas versões expandida e compacta, fiel à imagem, e
   nos estados de ausência; aprovação visual (inclui a nota da tabela, 15.7).
3) Mudança aditiva da RentCast (Comparables com [JsonIgnore] + método com desfecho), chave e
   helper compartilhado, ajuste da linha 107 do property-data-cache (se aprovado em 15.3);
   testes de não regressão de Properties, Integrations e NewLegalAnalysis.
4) Worker, options, registro no DI, projeções, PromptBuilder, PayloadMapper,
   modelo/guard/mapper/componente Angular, composição no diálogo.
5) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile, execução real e leitura do log.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
---------------------------------------------------------------------------------------------
15.1 RentCast: gravar UMA resposta real de /v1/avm/rent/long-term (imóvel da FL) com a chave
     já configurada e confirmar: presença e tipos de comparables[], valores reais de
     status/listingType, comportamento sem estimativa (corpo e status), e que o plano
     contratado permite exibir os comparáveis no relatório e guardá-los no pacote do Blob.
     Não bloqueia o passo 2.
15.2 Duplicidade "Asking price" com o Card 1 (⚠️ item 4). Recomendação: aprovar como exceção
     documentada (a imagem é o contrato e o preço é a base visível da relação), com mesma
     fonte e mesma precedência do Card 1, coberta por teste. Alternativa: "Asking price: see
     Card 1".
15.3 Consulta compartilhada da RentCast: autorizar a mudança de UMA linha no
     PropertyDataCacheWorker (linha 107) para usar SharedRentCastQueries, e o método com
     desfecho no IRentCastClient. Sem isso, o Card 14 funciona, mas uma execução pode fazer
     duas chamadas simultâneas à RentCast rent, contadas no teto diário.
15.4 Comparáveis de aluguel no payload do Card 14 e no Blob: confirmar com os termos da
     RentCast (15.1) a exibição e o armazenamento de endereços de anúncios de terceiros.
15.5 Uso de LLM no card: em aberto (arquiteto Bruno, PROMPT 14E). O worker entrega fatos e a
     relação calculada; o LLM só recebe ForReport.
15.6 Escopo: confirmar que a fase 1 é só o que a imagem mostra e que custos, cenários,
     restrições (por referência aos Cards 6, 7, 11 e 13) e FHA vão para a fase 2 com nova
     imagem.
15.7 Imagem de referência: salvar em legal-share (padrão CARD14_INVEST_IMG_REF.<ext>);
     aprovar a nota da tabela e o estado "No rental comparables returned" (6.1).
15.8 CSV: a RentCast não está mapeada no Fontes_API_Legal_analyses.csv, embora já seja fonte
     integrada do relatório. Recomendação: o mantenedor do CSV acrescentar a linha "Aluguel
     estimado e comparáveis de aluguel" (RentCast). Não altera este blueprint.
15.9 Base de código (reconferido em 2026-11-10): d_leevli-platform-api está em
     feat/legal-analysis-source-coordinator (57074dd6 "Modulariza Workers"), que contém o
     Card 5 e o Plano A; d_leevli-platform-web está em LEGAL (71e79913). Os Cards 6 a 10 e 13
     não estão implementados: peças compartilhadas (switch do PromptBuilder, helper de MLS de
     3.3) são criadas por quem for implementado primeiro; os outros só acrescentam.
15.10 Work item com critérios de aceite; branch feat/<card>-rental-investment; commits
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
