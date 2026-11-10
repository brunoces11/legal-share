# LEEVLI | LEGAL ANALYSIS | CARD 11 — PROPERTY TAXES AND RECURRING COSTS
# MASTER IMPLEMENTATION BLUEPRINT V1 — BASEADO NO TEMPLATE DO CARD 6 (CARD6_HOA_FINANCIAL.md, V3)
# (aderente ao código em 2026-11-10, incluindo o Plano A: coordenador de consultas por execução)

Este documento é o blueprint de implementação do Card 11 (Property Taxes and Recurring Costs).
Segue a estrutura do template do Card 6: as seções [REGRA FIXA] são idênticas às do template;
as seções [ESPECÍFICO DO CARD] são próprias do Card 11.

V1 (2026-11-10): a IMAGEM DE REFERÊNCIA do card é o contrato visual. Ela traz a versão
expandida e a compacta. Todo dado exibido está mapeado, na seção 6.1, para um campo real de
fonte programática conferida em 2026-11-10: RealEstateAPI Property Detail (taxInfo, a mesma
fonte já mapeada no Card 8) e os arquivos públicos do tax roll da Florida Department of
Revenue — NAL (valores e isenções, User's Guide 2026) e NAV (cobranças non-ad valorem,
navlayout.pdf). As seções [REGRA FIXA] foram reconferidas contra o código em 2026-11-10: as
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
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 11]
---------------------------------------------------------------------------------------------
Card dos IMPOSTOS e COBRANÇAS PÚBLICAS RECORRENTES do parcel: imposto publicado, valores do
tax roll (just, assessed e taxable), isenções e cobranças non-ad valorem. Todo dado vem de
fonte programática e carrega a origem e o ano. O card não estima o imposto do comprador, não
soma imposto e cobranças e não afirma situação de pagamento sem fonte.

Termos do card:
- Imposto publicado .... taxInfo.taxAmount da RealEstateAPI, do último release do assessor do
                         condado. Não é o tax bill nem o imposto do próximo ano.
- Tax roll ............. arquivo NAL (Name-Address-Legal) que o property appraiser envia à
                         Florida DOR; base de just value, assessed value e taxable value.
- School / non-school .. o roll separa valores para impostos do school district e para os
                         demais (DOR, campos 14–17); os dois podem diferir por limitações e
                         isenções.
- Cobrança pública ..... non-ad valorem assessment cobrada pelo método uniforme (F.S.
                         §197.3632), publicada no arquivo NAV (ex.: lixo, CDD).
- Folio ................ número do parcel (PARCEL_ID do NAL/NAV). Chave de consulta; não é
                         exibido (dono: Card 1).

O CSV mapeia quatro itens para este card (PROMPT 11C): "Avaliação fiscal" (Miami-Dade
Property Appraiser; Florida DOR), "Impostos e inadimplência" (Tax Collector), "Isenções e
portabilidade" (Property Appraiser; Tax Collector) e "CDD e distritos especiais" (Tax
Collector; registros do parcel; link DOR Data Portal). A imagem cobre os três primeiros em
parte e o quarto pela NAV.

### 6.1 CONTRATO VISUAL — IMAGEM DE REFERÊNCIA -> DADOS
Imagem recebida em 2026-11-10 com a versão expandida e a compacta. Marca do laboratório, não
dado: "Illustrative values" no rodapé (seção 12). O chip "Partial data" do cabeçalho É DADO:
é um estado calculado (regra abaixo).
Formato de cada linha: "campo (exemplo da imagem) <- fonte primária | alternativa | estado
quando ausente".

Cabeçalho
- Ícone de documento "TAX" + divisória + título "Property Taxes" em gradiente + subtítulo
  fixo "Tax Assessment & Public Charges" + chip de estado + seta.
- Chip de estado (Partial data) <- calculado no frontend a partir do view model:
  * "Partial data": pelo menos um campo do card está "Not available" ou "Not verified" e
    pelo menos um tem valor. Na fase 1 é o estado normal, porque Payment status é sempre
    "Not verified" (9.4);
  * "Data unavailable": nenhum campo com valor (todas as fontes indisponíveis);
  * sem chip: todos os campos com valor e verificados (só possível na fase 2).
  Texto e regra a aprovar com o HTML estático (15.6).

Painel 1 — "Reported property tax"
- Valor grande ($6,200) + legenda "Reported amount"
    <- RealEstateAPI /v2/PropertyDetail: data.taxInfo.taxAmount (string na documentação, ex.
       "12491.00"); moeda USD sem centavos quando inteiro
    |  cesta MLS: mlsDetail.homeDetails.taxAmount (gravação real "2342"), exibido com a
       legenda "Reported amount (as listed)" e limitação "taxAmountAsListed"
    |  "Not available"
- Assessor release year (2025)
    <- data.taxInfo.year — a documentação da RealEstateAPI define: "year => year of last Tax
       Assessor release (for that specific propertyInfo.address.county)"
    |  sem alternativa (o MLS não informa ano): "Not available" quando o valor veio do MLS
- Payment status (Not verified)
    <- nenhuma fonte integrada na fase 1 (Tax Collector, CSV "Impostos e inadimplência").
       Estado fixo "Not verified" com limitação "paymentStatusNotVerified". Nunca "Paid",
       "Current" ou "Delinquent" sem fonte do Tax Collector (fase 2). Flags agregadas de
       provedor (ex.: taxDelinquent em leadTypes) NÃO são usadas: CSV — "ausência de
       indicador agregado não comprova quitação".

Painel 2 — "Tax roll values" (subtítulo "Assessment year <ano>")
- Assessment year (2025) <- NAL ASMNT_YR (campo 4) do registro do parcel
- Just value ($485,000) <- NAL JV (campo 11)
- Assessed value - non-school ($420,000) <- NAL AV_NSD (campo 15)
- Taxable value - school ($395,000) <- NAL TV_SD (campo 16)
- Taxable value - non-school ($370,000) <- NAL TV_NSD (campo 17)
    Todos do MESMO registro NAL; sem alternativa (a RealEstateAPI publica assessedValue e
    marketValue sem separar school/non-school, e não são misturados no painel).
    |  painel inteiro "Not available" se o parcel não estiver no snapshot NAL.
    AV_SD (campo 14) não aparece na imagem e não é lido (15.6).

Painel 3 — "Exemptions & public charges"
- Homestead exemption ($50,000)
    <- NAL EXMPT_01 + EXMPT_02 do mesmo registro (DOR, campos 110–111: a primeira isenção
       homestead de até US$ 25.000 e a adicional, "196.031(1)(a)" e "(b)"). Soma
       determinística no backend; nenhuma das duas > 0 -> "None reported"
    |  snapshot NAL sem o parcel -> "Not available"
- Other exemptions (Not available)
    <- NAL: demais campos EXMPT_xx (fields 110–160) com valor > 0, exibidos como "<n>
       exemption(s) · $<total>"; nenhum -> "None reported"
    |  snapshot NAL sem o parcel -> "Not available" (estado do exemplo da imagem)
    A lista de quais campos são homestead/outros fica numa tabela estática versionada no
    código, com a URL do User's Guide da DOR do ano (gate 15.2).
- Public charges (2025) ($760)
    <- NAV Table N "Total assessments" (campo 6) do parcel; o ano do rótulo é "Tax year"
       (campo 5)
    |  parcel ausente do NAV com o NAV do condado carregado -> "None reported"
    |  NAV do condado não carregado -> "Not available"
- Linha de detalhe (Solid waste · F0001), uma por assessment do parcel (NAV Table D):
    <- "Function code" (D6, códigos 1–10 do formulário DR-503NA) traduzido pela tabela fixa
       do DR-503NA (gate 15.3) · "Levy identifier" (D4, ex. F0001); o valor de cada linha
       (D7 "Assessment amount", "Dollar amount of the assessment (Not the unit rate)") é
       exibido à direita quando houver mais de uma linha; até 5 linhas e "+<n> more".
       Código de função sem tradução conferida -> exibir "Function <código>".

Rodapé (três partes separadas por divisórias verticais)
- "Sources: <fontes que responderam>" com os nomes da imagem: "RealEstateAPI" e "Florida
  DOR". Texto fixo é proibido.
- Nota fixa em itálico (texto da imagem): "Public charges may be included in the reported
  tax; do not add twice."
- "Illustrative values" só no laboratório.

Versão compacta (está na imagem)
- Ícone + título | "Reported tax" / valor | "Release year" / ano | "Assessed - non-school" /
  valor | "Homestead exemption" / valor + seta para baixo. Rótulo pequeno em cima, valor em
  negrito embaixo, separados por divisórias. Ausente -> "Not available". O chip de estado não
  aparece na compacta (a imagem não o mostra).

Regra de leitura do contrato: cada campo sai com um único sourceId; os painéis 2 e 3 vêm do
snapshot da DOR do MESMO ano de roll; o painel 1 vem da RealEstateAPI. Nenhum total é
somado entre painéis.

### 6.2 Fases
Fase 1 (este blueprint, 1 PR/1 card) = exatamente a imagem:
- RealEstateAPI /v2/PropertyDetail (fonte COMPARTILHADA com o Card 8): imposto e ano.
- Cesta MLS: alternativa do imposto (homeDetails.taxAmount).
- Florida DOR NAL e NAV (arquivos públicos, por condado e ano): valores do roll, isenções e
  cobranças non-ad valorem. Fase 1 restrita a Miami-Dade (CSV); outros condados da FL entram
  carregando o arquivo do condado, sem mudar código.
Fase 2 (backlog, card/PR próprio):
- Tax Collector (situação de pagamento, ano de inadimplência, tax certificates, tax deed
  application): substitui "Not verified" em Payment status. Ver 6.3 sobre o Card 10.
- Portabilidade/Save Our Homes (NAL campos 104–109) e estimativa para o comprador (PROMPT
  11B): não estão na imagem.

### 6.3 Fronteira com outros cards (regra anti-duplicidade, ⚠️ item 4)
- Card 1: endereço, folio, proprietário, tipo e ano. O Card 11 usa o folio só como chave.
  Nenhum card atual exibe imposto, valores do roll, isenções ou cobranças (conferido em
  2026-11-10 em ui/legal-analysis/cards e data-access; a página do imóvel mostra
  assessedValue/taxAmount fora do relatório — property-detail.mapper.ts:116–119, 294–296 —
  e não conta para a regra).
- Card 6: taxa da associação. Card 14: usa imposto só por referência ("dono Card 11",
  CARD14_INVEST.md:403).
- Card 10 (CARD10_OWNERSHIP.md:460–461) atribui ao Card 11 tax liens, tax certificates e tax
  deeds. A imagem do Card 11 não os mostra: ficam na fase 2 junto com o Tax Collector (15.5).
- CDD/distritos especiais (CSV) são do Card 11 e aparecem como linhas de Public charges
  quando estiverem no NAV.

Proibições invariantes: sem "Paid", "Current", "Delinquent", "No taxes due" ou estimativa do
imposto do comprador; taxable value nunca apresentado como valor de mercado (CSV); just value
nunca apresentado como preço; isenção do vendedor nunca apresentada como transferível ao
comprador (CSV: "Benefício do vendedor não passa automaticamente ao comprador"); imposto e
cobranças nunca somados; null nunca vira 0; "None reported" só quando a fonte foi lida e não
tem o item; sem placeholder em produção.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM EM 2026-11-10) [ESPECÍFICO DO CARD 11]
---------------------------------------------------------------------------------------------
- LegalAnalysisWorkerIds (Application/Contracts/LegalAnalysisContracts.cs:126–130) declara só
  PropertyDataCache e FloodEnvironmentalRisk. Os demais cards existem só como blueprints em
  legal-share; quem for implementado primeiro cria o switch do PromptBuilder e o helper de
  MLS de 3.3.
- /v2/PropertyDetail NÃO está integrado (CARD8_FORECLOSURE.md, 7 e 9.2). O Card 8 define o
  cliente RealEstateApiPropertyDetailClient e prevê: "quando [o Card 10 ou 11] passar a
  usá-la, a fonte vira compartilhada: chave LegalAnalysisSourceKeys.PropertyDetail
  (externalSource, propertyId) e coordenador" (CARD8_FORECLOSURE.md:567).
- Cesta: mlsDetail.homeDetails.taxAmount existe (gravação real "2342",
  RealEstateApiOnlinePropertyDetailTests.cs:631); public.assessedValue existe
  (RealEstateApiPublicDto.cs:10), mas não é usado (não separa school/non-school).
- id do provedor para a PropertyDetail: `id` do item de mlsSearchDetail com conferência de
  listingId (mesma regra do Card 8, seção 8; FloodPropertyLocationResolver.cs:195–204 como
  referência).
- Florida DOR: nenhum código no repositório. Nenhum mecanismo de espelho de arquivo público
  existe ainda (o Card 6 previu um para o DBPR, gate 15.3 dele, não implementado).
- Frontend: rota card-flood (app.routes.ts:5–10) como padrão de laboratório; rawPayload é a
  união PropertyDataCacheWorkerPayload | FloodEnvironmentalRiskPayload
  (legal-analysis-result.model.ts:57); guard getFloodEnvironmentalRiskPayload
  (legal-analysis-worker-payload.ts:9–12); composição floodRiskCard em
  pages/listing-detail/components/legal-analysis-result-dialog (.ts:122; bloco @if em
  .html:64; Summary em .html:71).
- Imagem de referência do Card 11: AUSENTE em legal-share (recebida na conversa) -> 15.7.

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE E APLICABILIDADE [ESPECÍFICO DO CARD 11, SEGUINDO 3.3]
---------------------------------------------------------------------------------------------
- Resolução própria do Card 11, pelos mecanismos obrigatórios de 3.3: PropertyLookup +
  IPropertyDataCacheReader; com FetchMissingSources=true, MLS ausente buscado SOMENTE por
  SharedMlsQueries (DetailAsync/ListingAsync), compartilhando a consulta com os demais workers.
- Campos lidos: `id` do item de mlsSearchDetail (id do provedor), public.apn (folio),
  homeDetails.taxAmount, address.stateOrProvince, address.countyOrParish. Coordenadas NÃO
  são necessárias. Não usar FloodPropertyLocationResolver (3.3).
- Folio para a DOR: dígitos de public.apn; também lotInfo.apn/apnUnformatted da PropertyDetail
  quando a resposta chegar. Iguais -> usa; diferentes -> não consulta a DOR ("folioMismatch");
  ausente -> "folioUnavailable". Igualdade com PARCEL_ID do NAL: comparar só dígitos (formato
  do PARCEL_ID de Miami-Dade confirmado no gate 15.1).
- Aplicabilidade (pré-requisito, 4.1):
  * FL -> DOR aplicável; condado com snapshot carregado -> consulta; sem snapshot ->
    coverageMissing ("dorRollNotLoaded");
  * fora da FL -> DOR notApplicable; a PropertyDetail continua (cobertura nacional);
  * sem id do provedor -> PropertyDetail não consultada ("providerPropertyIdUnavailable").
- Número do condado da DOR (CO_NO) pela tabela da DOR (Miami-Dade = 23 no User's Guide;
  conferir no gate 15.1), escolhido a partir de countyOrParish normalizado como no
  MiamiDadeEvacuationZoneClient.

---------------------------------------------------------------------------------------------
## 9. FONTES DA FASE 1 [ESPECÍFICO DO CARD 11]
---------------------------------------------------------------------------------------------
### 9.1 Cesta MLS — pré-requisito e alternativa do imposto, fonte COMPARTILHADA
- Leitura pela cesta, sem alterar nenhum DTO; busca quando ausente só via SharedMlsQueries
  (3.2). Proibido chamar IOnlinePropertyDetail direto.
- homeDetails.taxAmount: número finito > 0 (string ou número); senão ausente.
### 9.2 RealEstateAPI /v2/PropertyDetail — conteúdo, fonte COMPARTILHADA (Cards 8 e 11)
- Mesmo endpoint, cliente e contrato do Card 8 (CARD8_FORECLOSURE.md, 9.2 e 11): POST
  https://api.realestateapi.com/v2/PropertyDetail com { "id": <id do provedor> }.
- Campos usados pelo Card 11 (documentação "Property Detail Response Object", lida em
  2026-11-10): data.taxInfo.taxAmount, data.taxInfo.year, data.lotInfo.apn,
  data.lotInfo.apnUnformatted. assessedValue/marketValue/assessmentYear NÃO são exibidos
  (o painel 2 é do NAL).
- Fonte usada por 2 workers -> OBRIGATÓRIO passar pelo coordenador (3.2) com a chave
  LegalAnalysisSourceKeys.PropertyDetail(externalSource, propertyId); assim Card 8 e Card 11
  dividem UMA chamada (1 crédito) por execução. O contrato do cliente já separa falha
  (exceção) de resultado válido (found | notFound), condição de 3.2.
- Quem implementar primeiro (Card 8 ou 11) cria o cliente e a chave; o outro só reutiliza
  (15.4). Sem cache próprio; leitura tolerante por JsonElement.
- Estado: found = taxAmount válido; noResults = imóvel sem taxInfo/taxAmount; 404 ->
  noResults "propertyNotFound"; falha/timeout -> unavailable.
### 9.3 Florida DOR — NAL e NAV (conteúdo; arquivos públicos por condado) — EXCLUSIVA
- Fonte oficial: Florida Property Tax Data Portal (floridarevenue.com/property/Pages/
  DataPortal.aspx -> "Tax Roll Data Files": pastas NAL, NAP, NAV, SDF). Documentação lida em
  2026-11-10: "2026 User's Guide — Department Property Tax Data Files" (NAL) e
  NAV/navlayout.pdf. A DOR informa que publica só a versão mais atual de cada roll; versões
  anteriores são por pedido; os rolls chegam em 1º de julho (preliminar), outubro (final
  inicial) e após a certificação (final pós-VAB).
- NAL — campos usados: CO_NO (1), PARCEL_ID (2), ASMNT_YR (4), JV (11), AV_NSD (15), TV_SD
  (16), TV_NSD (17), EXMPT_01–EXMPT_82 (110–160). Campos de proprietário, endereço, venda e
  legal NÃO são lidos nem guardados (OWN_NAME, PHY_ADDR1 etc.; dono: Cards 1 e 10).
- NAV (método uniforme, F.S. §197.3632), CSV sem cabeçalho:
  * Table N (arquivo NAVN<cc><aa><ss>.TXT): Roll type, County number, PA Parcel Number, TC
    Account number, Tax year, Total assessments, Number of assessments, Tax roll sequence;
  * Table D (NAVD...TXT): Record type, County number, PA Parcel Number, Levy identifier,
    Local government code, Function code (1–10 do DR-503NA), Assessment amount, sequence.
  TC Account number não é lido.
- Não é API: são arquivos por condado e ano. NUNCA baixar por execução. Espelho local
  versionado, no padrão previsto pelo Card 6 para o DBPR: snapshot por condado com URL de
  origem, roll (preliminar/final), data de publicação, data de download e hash; só as colunas
  acima; atualização manual na fase 1 e rotina de verificação na fase 2. Forma de
  armazenamento e consulta (o NAL de Miami-Dade tem perto de um milhão de parcels, grande
  demais para memória) -> decisão 15.1. Sem migration/tabela no banco do produto (regra
  [REGRA FIXA] da seção 2).
- Atualidade: o payload grava o ano do roll, o tipo do roll e a data do snapshot; o rodapé
  cita "Florida DOR" só quando o parcel foi encontrado.
- Estado: found = parcel no NAL (painéis 2 e 3) e/ou no NAV; noResults = snapshot carregado
  sem o parcel; coverageMissing = condado sem snapshot; unavailable = falha de leitura.
### 9.4 Situação de pagamento — SEM FONTE na fase 1
- O CSV indica o Tax Collector do condado; não há API integrada nem conferida. Payment status
  sai sempre "Not verified" (6.1). Não entra em sourceStatus como fonte consultada; entra só
  como limitação.
### 9.5 Florida Statutes — NÃO exibidos pelo Card 11
- O CSV cita F.S. §§193.011, 193.114, 197.122, 197.432, 197.502, 196.031, 193.155, 190.021 e
  197.3632. A imagem não tem bloco de base legal: o card não exibe estatutos e não cria
  entradas no registro único (3.4) na fase 1.

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD (worker "property-taxes", schemaVersion 1) [ESPECÍFICO]
---------------------------------------------------------------------------------------------
{
  schemaVersion, collectedAtUtc,
  reportedTax: { status, sourceId?, amount?:{ value, currency:"USD" }, assessorReleaseYear?,
                 asListed: bool },
  paymentStatus: { status:"notVerified" },
  taxRoll: { status, sourceId?, assessmentYear?, rollType?:"preliminary"|"initialFinal"|
             "final", snapshotDate?, justValue?, assessedValueNonSchool?,
             taxableValueSchool?, taxableValueNonSchool? },
  exemptions: { status, sourceId?, homestead?:{ value, fields[] },
                other?:{ count, total } },
  publicCharges: { status, sourceId?, taxYear?, total?,
                   items[]:{ functionCode, functionLabel?, levyIdentifier, amount? } },
  sourceStatus[]: formato padrão (4.1),
  limitations[]
}
- sourceId: "realestateapi-property-detail" | "realestateapi-mls" | "florida-dor-nal" |
  "florida-dor-nav". status das seções: valores de FloodSourceStatus (4.1). Valores em USD
  inteiros como publicados (o NAL não tem centavos; taxAmount pode ter).
- Status do worker (regra 4.1): fontes de conteúdo = PropertyDetail, MLS (só para o
  imposto), NAL e NAV. Alguma found -> dataAvailable; senão alguma noResults -> noResults;
  senão -> sourceUnavailable. Payment status não é fonte e não define status.
- Limitações (códigos fixos): paymentStatusNotVerified (sempre na fase 1),
  publicChargesMayBeInReportedTax (sempre que houver os dois), taxAmountAsListed,
  taxableValueNotMarketValue (sempre que houver roll), exemptionsNotTransferable (sempre que
  houver isenção), rollYearDiffersFromReleaseYear, dorRollNotLoaded, folioUnavailable,
  folioMismatch, providerPropertyIdUnavailable, payloadReduced.
- Sem payload bruto, sem folio, sem proprietário, sem endereço. Payload alvo <= 6 KB
  (MaxPayloadBytes 20.000); parte reduzida primeiro: publicCharges.items (corte para 10),
  com "payloadReduced".
- Projeções (classe estática PropertyTaxesProjection):
  * Read: valida schemaVersion, enums, faixas numéricas (>= 0) e coleções;
  * ForApi: payload público do card;
  * ForReport: resumo para o LLM — valores, anos, tipo do roll, homestead (bool + valor),
    contagem/total de outras isenções, total e funções das cobranças, sourceStatus,
    limitations. Sem levy identifiers, sem folio.

---------------------------------------------------------------------------------------------
## 11. BACKEND [ESPECÍFICO DO CARD 11, SEGUINDO 3, 4 E 5]
---------------------------------------------------------------------------------------------
- Application/Contracts/PropertyTaxesContracts.cs: payload tipado + porta
  IFloridaTaxRollSource (GetParcelAsync(countyNumber, parcelDigits) -> NAL e NAV do parcel,
  metadados do snapshot; Found | NotFound | NotLoaded; falha por exceção). A porta da
  PropertyDetail é a do Card 8 (reutilizada, não duplicada).
- Application/Contracts/PropertyTaxesProjection.cs: Read, ForApi, ForReport.
- Application/Options/PropertyTaxesOptions.cs: { SourceTimeout=8 s, TotalBudget=15 s,
  MaxPayloadBytes=20.000, MaxPublicChargeItems=10 }; validação no
  NewLegalAnalysisModuleDependencyInjection (SourceTimeout <= TotalBudget; TotalBudget <
  WorkerTimeout; MaxPayloadBytes < limite geral), conforme 4.1.
- Tabelas estáticas versionadas (sem banco, sem DI), com URL da fonte e data de conferência:
  campos de isenção homestead do NAL (EXMPT_01, EXMPT_02) e rótulos das funções 1–10 do
  DR-503NA (gate 15.3).
- Integrations.Infrastructure/FloridaDor/: leitor do snapshot (implementação de
  IFloridaTaxRollSource) e utilitário de carga manual do snapshot (fora do request), conforme
  a decisão 15.1. Options FloridaDorTaxRollOptions { Enabled, SnapshotLocation } com
  ValidateOnStart. Sem ProjectReference novo (já existe).
- RealEstateApiPropertyDetailClient e LegalAnalysisSourceKeys.PropertyDetail: criados pelo
  card que chegar primeiro (Card 8 ou 11), como em 9.2.
- Infrastructure/Workers/PropertyTaxesWorker.cs:
  Id = LegalAnalysisWorkerIds.PropertyTaxes ("property-taxes"); registro explícito em
  AddNewLegalAnalysisWorkers. Sequência: identidade (seção 8) -> PropertyDetail (pelo
  coordenador) e DOR em paralelo, cada um isolado com timeout -> montagem pelas regras de 6.1
  -> limite de payload. Status pela regra 4.1. TimeProvider; ILogger + ILegalAnalysisRunTrace
  com RunId, sourceId, status, durationMs e ano do roll; nunca folio, proprietário ou payload.
- LegalAnalysisReportPromptBuilder: switch por WorkerId (seção 5, item 5) com PropertyTaxes
  -> PropertyTaxesProjection.ForReport; demais inalterados. O LLM nunca estima imposto,
  soma cobranças nem afirma pagamento.
- LegalAnalysisWorkerPayloadMapper: acrescentar PropertyTaxes -> PropertyTaxesProjection.
  ForApi, mantendo os casos existentes.
- Não alterar: coordenador (só nova chave em LegalAnalysisSourceKeys), orquestrador, cesta,
  clientes MLS/RentCast, DTOs MLS, persistência, contratos HTTP, outros workers.

---------------------------------------------------------------------------------------------
## 12. FRONTEND (Angular) [ESPECÍFICO DO CARD 11, SEGUINDO 4.2 E 5]
---------------------------------------------------------------------------------------------
- data-access/models/legal-analysis-taxes-payload.model.ts (payload) e
  legal-analysis-taxes.model.ts (view model).
- legal-analysis-result.model.ts: PropertyTaxesPayload na união de rawPayload.
- legal-analysis-worker-payload.ts: getPropertyTaxesPayload (workerId 'property-taxes' +
  schemaVersion 1), no padrão de getFloodEnvironmentalRiskPayload.
- data-access/legal-analysis-taxes.mapper.ts + .spec + .fixture: payload -> view model
  (moeda en-US sem centavos para inteiros, anos, "None reported"/"Not available", linhas de
  cobranças, chip de estado de 6.1, segmentos da compacta).
- ui/legal-analysis/cards/property-taxes/: standalone, signals, colapsável (seção 5, item 11).
- Layout expandido — a imagem de referência manda (PROMPT, Regra 7):
  1) Cabeçalho: ícone de documento "TAX" (SVG, aria-hidden) + divisória vertical + título em
     gradiente com subtítulo cinza abaixo; à direita, chip de estado com borda azul e seta.
  2) Três painéis com borda e cabeçalho em faixa clara, larguras ~30% / 32% / 34%:
     painel 1: título; valor grande em azul; legenda "Reported amount"; linha "Assessor
     release year <ano em negrito>"; divisória; "Payment status" à esquerda e valor em
     negrito à direita;
     painel 2: título + subtítulo "Assessment year <ano em negrito>"; quatro linhas rótulo à
     esquerda e valor em negrito à direita, com divisórias;
     painel 3: título; linhas Homestead exemption, Other exemptions e "Public charges
     (<ano>)" com valor em negrito à direita; linha(s) de detalhe "<função> · <levy>" sem
     negrito.
  3) Rodapé sob uma linha divisória, três partes com divisórias verticais: "Sources:" em
     negrito + fontes; nota em itálico centralizada; "Illustrative values" em negrito à
     direita (só laboratório).
- Linha compacta (está na imagem): segmentos de 6.1.
- Responsividade: até 900 px os três painéis empilham e o rodapé vira três linhas; até 600 px
  a compacta quebra em grade 2×2 sob o título.
- Rota de teste: features/listings/pages/card-taxes (path "card-taxes", fora do Shell,
  canMatch !environment.production, loadComponent), no padrão de pages/card-flood. Mock com os
  valores da imagem.
- Composição no diálogo: computed taxesCard + bloco @if após o bloco do card de número
  anterior já implementado (Card 10, 9, 8, 7, 6 ou 5) e antes do Summary.
- Regras ESLint de template do AGENTS.md; a11y teclado/leitor de tela (chip com texto, nunca
  só cor). Sem PDF na fase 1.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE
---------------------------------------------------------------------------------------------
Backend (unit): imposto pela PropertyDetail, pelo MLS (legenda "as listed", ano "Not
available") e ausente; taxAmount string com centavos e inválido; NAL encontrado, ausente e
condado sem snapshot; homestead = EXMPT_01 + EXMPT_02 (um, os dois, nenhum -> "None
reported"); outras isenções (0, 1, N); NAV com 0, 1 e N assessments e função sem rótulo;
"None reported" x "Not available"; folio por apn e por lotInfo, iguais e divergentes; fora da
FL; sem id do provedor; PropertyDetail pelo coordenador (uma chamada para Cards 8 e 11 na
mesma execução) e sem coordenador; orçamento/parcial com TimeProvider fake; isolamento de
fonte; limite de payload; projeções Read/ForApi/ForReport; payload mapper; prompt sem folio
nem levy identifier; Payment status nunca diferente de notVerified.
Backend (reaproveitamento): MLS por SharedMlsQueries; nenhuma chamada direta a
IOnlinePropertyDetail; leitura pela cesta antes de buscar; nenhum uso de
FloodPropertyLocationResolver; cliente da PropertyDetail único entre Cards 8 e 11.
Backend (integração): snapshot real reduzido da DOR (amostra de Miami-Dade com NAL e NAV) e
fixture gravada da PropertyDetail; worker no orquestrador sem afetar property-data-cache e
Flood; WebApiBoundaryTests e suíte completa.
Frontend: specs do mapper (moeda, anos, estados, chip) e do componente (expandido/compacto/
parcial/sem dados), guards; nenhum texto "Paid", "Delinquent" ou "No taxes due"; smoke
desktop/mobile com capturas no work item, comparadas com a imagem de referência.
Homologação real: casa homestead em Miami-Dade com cobrança de lixo; condomínio não
homestead; imóvel da FL fora de Miami-Dade (só PropertyDetail); imóvel fora da FL. Conferir no
log/LegalAnalysis.txt os eventos Worker e SourceQuery (a PropertyDetail deve aparecer como
Joined quando o Card 8 já a iniciou).
Aceite: todos os campos da imagem dinâmicos e com fonte; cards já implementados e Summary
inalterados; nenhum dado de outro card repetido; nenhum valor simulado; nenhuma soma entre
imposto e cobranças; Payment status sempre "Not verified" na fase 1; execução total < 120 s.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO
---------------------------------------------------------------------------------------------
1) Validação de dados e fontes: este documento + conferência anti-duplicidade + decisões
   15.1, 15.4 e 15.6 + gates 15.2 e 15.3.
2) Rota card-taxes com HTML estático nas versões expandida e compacta, fiel à imagem;
   aprovação visual.
3) Worker com a PropertyDetail (pelo coordenador) e a alternativa do MLS: painel 1 completo;
   painéis 2 e 3 "Not available" com "dorRollNotLoaded". Options, registro no DI, projeções,
   PromptBuilder, PayloadMapper, modelo/guard/mapper/componente Angular, composição.
4) Snapshot da DOR (NAL + NAV de Miami-Dade) como incremento aditivo, quando 15.1–15.3
   estiverem resolvidos: preenche os painéis 2 e 3 no mesmo payload; o card não muda de forma.
5) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile, execução real e leitura do log.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
---------------------------------------------------------------------------------------------
15.1 DOR — armazenamento do snapshot: os arquivos NAL/NAV são por condado e grandes (Miami-
     Dade perto de um milhão de parcels), sem API. Decidir onde guardar o snapshot reduzido às
     colunas de 9.3 e como consultá-lo por parcel sem migration/tabela no banco do produto
     (ex.: arquivo indexado por condado/ano no Blob ou no disco do serviço), e quem o atualiza
     a cada roll (julho, outubro, final). Baixar e conferir o arquivo real de Miami-Dade:
     número do condado (CO_NO), formato do PARCEL_ID frente ao folio do MLS e tamanho.
     Bloqueia o passo 4.
15.2 Isenções: confirmar no User's Guide do ano do roll quais EXMPT_xx compõem "Homestead
     exemption" (hoje: EXMPT_01 e EXMPT_02) e se a soma é a forma desejada de exibição.
15.3 NAV: obter a tabela de Function codes 1–10 do formulário DR-503NA (Tax Collector's
     Report on Non-Ad Valorem Assessments) para os rótulos (ex.: "Solid waste") e confirmar o
     formato do Levy identifier (ex. da imagem: F0001) em um NAVD real de Miami-Dade.
15.4 PropertyDetail compartilhada com o Card 8: confirmar que o card implementado primeiro
     cria cliente e chave do coordenador, e o custo (1 crédito por execução, dividido entre os
     dois cards). Gate 15.1 do Card 8 (resposta real) vale também para taxInfo.
15.5 Payment status e tax liens/certificates/deeds: Tax Collector de Miami-Dade sem API
     conferida -> fase 2. Registrar no Card 10 que a referência a eles depende dessa fase.
15.6 Chip de estado ("Partial data" / "Data unavailable" / sem chip) e uso de AV_SD (não
     exibido): aprovar com o HTML estático.
15.7 Imagem de referência: salvar em legal-share (padrão CARD11_TAXES_IMG_REF.<ext>).
15.8 Uso de LLM no card: em aberto (arquiteto Bruno, PROMPT 11E). O worker entrega fatos; o
     LLM só recebe ForReport.
15.9 Base de código (reconferido em 2026-11-10): d_leevli-platform-api está em
     feat/legal-analysis-source-coordinator (57074dd6 "Modulariza Workers"), que contém o
     Card 5 e o Plano A; d_leevli-platform-web está em LEGAL (71e79913). A branch do card
     parte da branch de integração que contenha Card 5 + Plano A.
15.10 Work item com critérios de aceite; branch feat/<card>-property-taxes; commits
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
