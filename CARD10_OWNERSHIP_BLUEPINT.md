# LEEVLI | LEGAL ANALYSIS | CARD 10 — OWNERSHIP AND PUBLIC RECORDS
# MASTER IMPLEMENTATION BLUEPRINT V1 — BASEADO NO TEMPLATE DO CARD 6 (CARD6_HOA_FINANCIAL.md, V3)
# (aderente ao código em 2026-11-10, incluindo o Plano A: coordenador de consultas por execução)

Este documento é o blueprint de implementação do Card 10 (Ownership and Public Records).
Segue a estrutura do template do Card 6: as seções [REGRA FIXA] são idênticas às do template;
as seções [ESPECÍFICO DO CARD] são próprias do Card 10.

V1 (2026-11-10): a IMAGEM DE REFERÊNCIA do card é o contrato visual. Ela traz a versão
expandida com registros, a compacta e a expandida sem registros. Todo dado exibido está
mapeado, na seção 6.1, para um campo real da Miami-Dade Clerk Official Records API, a mesma
fonte já mapeada no Card 8 (CARD8_FORECLOSURE.md, 9.3), conferida em 2026-11-10 na
documentação da API e no layout oficial dos arquivos de Official Records do Clerk. As seções
[REGRA FIXA] foram reconferidas contra o código em 2026-11-10: as referências de arquivo e
linha datadas de 2026-11-06 continuam válidas.

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
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 10]
---------------------------------------------------------------------------------------------
Card do ÍNDICE DE REGISTROS OFICIAIS do imóvel: os instrumentos de transmissão e de
financiamento gravados no condado (escritura, hipoteca, cessão, quitação/liberação), com
data de registro, partes e referência do Clerk. O card mostra o ÍNDICE publicado pelo Clerk;
não lê o conteúdo dos documentos, não certifica título, não calcula saldo e não afirma que
uma hipoteca está quitada ou em aberto.

Termos do card:
- Instrumento ........ documento gravado nos Official Records (ex.: deed, mortgage).
- Índice ............. linhas que o Clerk publica para cada instrumento (tipo, datas, partes,
                       referências). Uma linha por parte (PARTY_SEQUENCE).
- CFN ................ Clerk's File Number: ano + "R" + sequência (layout oficial: "02R000001";
                       ano gravado com 4 dígitos).
- Parte direta (D) ... parte do lado "direto" do índice (FIRST_PARTY com PARTY_CODE D).
- Parte reversa (R) .. parte do lado "reverso" do índice (PARTY_CODE R). O layout oficial
                       exemplifica a hipoteca: devedor = direta, banco = reversa.
- Registro relacionado  instrumento original citado por outro (ex.: a quitação cita a
                       hipoteca por ORIG_CFN_* ou ORIG_REC_BOOK/PAGE), só quando o Clerk o
                       publica.
- Folio .............. número do parcel no condado (Miami-Dade: 13 dígitos). Chave da busca;
                       não é exibido (dono: Card 1).

### 6.1 CONTRATO VISUAL — IMAGEM DE REFERÊNCIA -> DADOS
Imagem recebida em 2026-11-10, com três estados: expandido com registros, compacto e
expandido sem registros. O chip "Illustrative data" e o sufixo "· Illustrative values" do
rodapé são marcas do laboratório, não dados (seção 12).
Fonte única de todos os dados: Miami-Dade Clerk Official Records API, GET
api/OfficialRecords?parameter1={folio}&parameter2=FN (9.2).
Formato de cada linha: "campo (exemplo da imagem) <- campo da API | regra".

Cabeçalho
- Ícone de documento + título "Ownership & Public Records" (grafia da imagem, com "&") +
  subtítulo fixo "Official record index" + barra decorativa em gradiente + seta. Sem dado.

Tabela "Instrument · Recorded · Parties · Clerk reference" (estado com registros)
- Cada linha da tabela = UM instrumento: as linhas da API são agrupadas por CFN_YEAR +
  CFN_SEQ (a API devolve uma linha por parte). Ordem cronológica crescente por REC_DATE;
  desempate pela sequência do CFN (como na imagem: 2020, 2020, 2023).
- Instrument (Warranty Deed · Mortgage · Satisfaction / Release)
    <- DOC_TYPE (código do condado; o layout oficial dá "MOR" = "MORTGAGE"), traduzido por
       uma tabela fixa código -> rótulo montada com a lista oficial de códigos do Clerk
       (gate 15.1). Rótulo da família quando o Clerk não distingue subtipo.
    REGRA: "Warranty Deed" só aparece se o Clerk publicar código próprio para esse subtipo.
       Se o índice só tiver "deed", o rótulo é "Deed". O subtipo nunca é inferido de outra
       fonte (o rodapé da imagem só cita o Clerk) — decisão 15.4.
- Recorded (Jan 10, 2020)
    <- REC_DATE (data de calendário, sem hora e sem conversão de fuso; formato "MMM d, y").
- Parties (Seller → ABC Florida Holdings LLC)
    <- FIRST_PARTY das linhas com PARTY_CODE "D" (lado esquerdo) "→" FIRST_PARTY das linhas
       com PARTY_CODE "R" (lado direito); SECOND_PARTY (nome da contraparte) só confirma o
       par. É exatamente a convenção D -> R do layout oficial, e reproduz as três linhas da
       imagem: escritura vendedor -> comprador; hipoteca devedor -> credor; quitação
       credor -> devedor (confirmar com respostas reais, gate 15.1).
    Privacidade (interpretação da imagem, decisão 15.3): a imagem mostra "Seller" no lugar do
       nome do vendedor e mostra os nomes das empresas. Regra proposta: nome exibido só
       quando o índice marcar a parte como empresa (FIRM_INDIV, valores no gate 15.1); pessoa
       física ou marcação ausente -> rótulo do papel pela família e pelo lado:
         deed ............. D "Seller"   · R "Buyer"
         mortgage ......... D "Borrower" · R "Lender"
         assignment ....... D "Assignor" · R "Assignee"
         satisfaction ..... D "Lender"   · R "Borrower"
       O nome da pessoa física não entra no payload, no Blob, no LLM nem no log.
    Mais de 2 partes no mesmo lado: as 2 primeiras + "+<n>". Lado vazio -> "Not indexed".
- Clerk reference (CFN 2020-00123456)
    <- CFN_YEAR + CFN_SEQ, no formato oficial do Clerk "CFN <ano>R<sequência>". A imagem do
       Card 10 mostra "2020-00123456" e a do Card 8 mostra "2024R-123456": as duas são
       ilustrativas e divergem entre si. Os dois cards usam UM formatador compartilhado no
       frontend, com o formato confirmado no gate 15.1. Sem CFN -> "BK <REC_BOOK> PG
       <REC_PAGE>"; sem os dois -> "Not available".

Linha abaixo da tabela — "Related record: CFN 2020-00123457 · when supplied" + "Related
record is a reference only, not automatic proof of discharge."
- <- ORIG_CFN_YEAR + ORIG_CFN_SEQ do instrumento que cita outro; sem CFN original,
     ORIG_REC_BOOK + ORIG_REC_PAGE. LINK_DOCTYPE (tipo do original) só para o rótulo.
- Uma linha por instrumento que cita um original: "Related record: <referência>"; quando há
  mais de um instrumento citando, a linha começa pelo instrumento que cita
  ("Satisfaction / Release · Related record: ..."). Quando a referência coincide com uma
  linha da tabela, as duas ficam associadas (aria-describedby); nunca se associa por nome,
  valor ou data.
- "· when supplied" é lido como anotação de design ("a linha só aparece quando o Clerk
  publica a referência"), não como texto renderizado — confirmar na aprovação visual (15.7).
- O aviso "Related record is a reference only, not automatic proof of discharge." é texto
  fixo e aparece sempre que houver ao menos uma linha de registro relacionado (CSV
  "Hipotecas registradas": "Ausência de baixa exige esclarecimento").
- ORIG_MISC_REF nunca é exibido (pelo layout oficial pode conter endereço, valor ou número
  de processo).

Rodapé — "Source: Miami-Dade Clerk Official Records API"
- Montado a partir de sourceStatus: aparece só quando a fonte respondeu (com ou sem
  registros). Sem consulta, o rodapé não cita a fonte.

Versão compacta (está na imagem)
- "Ownership & Public Records | <n> indexed records" + seta para baixo, com n = número de
  instrumentos no escopo encontrados. Proposta para os demais estados: "No indexed records
  found" (consulta sem registros) e "Records unavailable" (sem consulta ou falha).

Estado sem registros (está na imagem)
- Ícone de documento com lupa + "No transfer records found in this search" + "Other
  recorded documents may exist." + rodapé com a fonte. Só quando a consulta respondeu e não
  havia instrumento no escopo (noResults). Nunca usado para falha ou falta de cobertura.

Estado indisponível (NÃO está na imagem; proposta para aprovar com o HTML estático, 15.7)
- Mesmo layout do estado vazio, com "Official records not available for this property" e
  uma linha de motivo: "County records for this county are not yet integrated" |
  "Parcel number unavailable" | "The Clerk records service did not respond". Sem rodapé de
  fonte. Mostrar "No transfer records found" nesses casos seria afirmar uma busca que não
  ocorreu.

Regra de leitura do contrato: nenhum valor da tabela é fixo no card definitivo; são fixos
só rótulos, a tabela código -> rótulo, os rótulos de papel e os dois textos de aviso.

### 6.2 Fases
Fase 1 (este blueprint, 1 PR/1 card) = exatamente a imagem ("transfer records"):
- Miami-Dade Clerk Official Records por folio, só imóveis em Miami-Dade.
- Famílias de instrumento no escopo: deed (todos os códigos de escritura), mortgage
  (inclusive modificação), assignment of mortgage e satisfaction/release of mortgage. Lista
  de códigos no gate 15.1.
- Fora de Miami-Dade, sem folio ou com a fonte indisponível: estado indisponível (6.1).
Fase 1b (incremento recomendado, decisão 15.5): liens, claims of lien, judgments e suas
liberações na MESMA tabela, mesma fonte e mesma consulta, sem chamada nova. Exige trocar o
texto do estado vazio ("No indexed records found in this search"), que hoje fala em
"transfer records". Inclui os liens de condomínio/HOA (§718.116/§720.3085) que o blueprint do
Card 6 atribuiu ao Card 10.
Fase 2 (backlog, card/PR próprio): Clerks de outros condados (Broward primeiro); descrição
legal da escritura, easements e parking/storage (dependem do conteúdo do documento, não do
índice); imagens dos documentos (RecordImages da API); flags `judgment`/`lien` da
RealEstateAPI PropertyDetail e Involuntary Liens API (add-on pago) como fonte cruzada;
"Compatibilidade entre vendedor do contrato e proprietário registrado" (Pós-oferta, Card 15).

### 6.3 Fronteira com outros cards (regra anti-duplicidade, ⚠️ item 4)
- Card 1 (Property Identity) já EXIBE "Registered owner", "Parcel ID", "Last sale price" e
  "Last sale date" (legal-analysis-property-card.mapper.ts:85–89, 123–124, 128–129, 134,
  136), todos vindos da RealEstateAPI. O Card 10 NÃO exibe proprietário atual, folio, preço
  nem data de venda. Exibe a data de REGISTRO e as partes de cada instrumento, que são dado
  do índice do Clerk. A data de registro de uma escritura pode ficar perto da "Last sale
  date" do Card 1, mas são campos e fontes diferentes: não há repetição do mesmo dado.
- Card 8 (Foreclosure) é o DONO do lis pendens (CARD8_FORECLOSURE.md, 6.3 e 15.5). Os
  instrumentos de lis pendens e suas liberações ficam FORA da tabela do Card 10. A imagem
  não traz linha "Lis pendens: see Card 8"; o card não a acrescenta.
- Card 8 e Card 10 leem a MESMA resposta (Official Records por folio), com filtros
  diferentes: o Card 8 usa as linhas de lis pendens, o Card 10 usa as de transmissão e
  financiamento. A fonte passa a ser COMPARTILHADA (9.2).
- Card 6: dados financeiros da associação. Card 11: tax liens, tax certificates e tax deeds
  (o Tax Collector é a fonte; tax deed gravado aparece como deed no índice e entra na
  tabela pelo código do Clerk, sem rótulo fiscal). Card 12: notices of commencement e
  permits (fora do escopo de instrumentos). Card 15: título, title commitment e contrato.
- Valores financeiros do índice (CONSIDERATION_1/2, DEED_DOC_TAX, DOCUMENTARY_STAMPS,
  INTANGIBLE, SURTAX) não são exibidos (a imagem não os mostra) e não entram no payload.
- Conferido em 2026-11-10: nenhum card atual (1, 3, 4, 5) exibe instrumentos registrados,
  CFN ou partes de instrumentos.

Proibições invariantes: sem score, "Clear title", "Free and clear", "Open mortgage",
"Paid off", "No liens" ou qualquer conclusão sobre título; registro relacionado nunca é
apresentado como prova de quitação; ausência de quitação nunca vira "mortgage open";
ausência de registro nunca vira "sem instrumentos" (só "No transfer records found in this
search" + "Other recorded documents may exist."); saldo de hipoteca, valores e impostos do
índice não são exibidos; partes nunca são inferidas de outra fonte; nome de pessoa física
nunca é exibido, gravado, enviado ao LLM ou logado (15.3); sem placeholder em produção.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM EM 2026-11-10) [ESPECÍFICO DO CARD 10]
---------------------------------------------------------------------------------------------
- LegalAnalysisWorkerIds (Application/Contracts/LegalAnalysisContracts.cs:126–130) declara só
  PropertyDataCache e FloodEnvironmentalRisk. Os Cards 6, 7, 8 e 9 NÃO estão implementados
  (só blueprints em legal-share). Quem for implementado primeiro cria o switch do
  PromptBuilder (seção 5, item 5) e o helper de MLS de 3.3.
- Miami-Dade Clerk: nenhum cliente no codebase. O Card 8 (CARD8_FORECLOSURE.md, seção 11)
  especifica IMiamiDadeClerkRecordsClient.GetOfficialRecordsByFolioAsync e
  MiamiDadeClerkOptions; o Card 10 usa a MESMA porta e as MESMAS options. A pasta
  Integrations.Infrastructure/MiamiDade já existe (MiamiDadeEvacuationZoneClient, Card 5).
- Folio na cesta: RealEstateApiPublicDto.Apn (linha 7), lido em mlsSearchDetail.public.apn.
  Fixture com formato do condado: "01-4210-018-0400" (13 dígitos;
  RealEstateApiMlsSearchItemMapperTests.cs:362). O Card 1 o exibe como "Parcel ID".
- Condado: address.countyOrParish do registro MLS (lido em FloodPropertyLocationResolver.cs:
  60). A normalização do nome de Miami-Dade existe em MiamiDadeEvacuationZoneClient (linhas
  16–20).
- Nenhum DTO do codebase tem instrumento, CFN ou parte de instrumento. RealEstateApiPublicDto
  tem RecordingDate (linha 39), LastSaleDate e valores de hipoteca: não são usados pelo Card 10.
- ConfigureEnvironmentalHttpClient (IntegrationsDependencyInjection.cs:419–426) aplica
  RemoveAllLoggers: requisito para o AuthKey do Clerk, que vai na query string.
- LegalAnalysisSourceKeys (Infrastructure/Orchestration/LegalAnalysisSourceKeys.cs) declara só
  MlsDetail e MlsListing.
- Frontend: rota card-flood (app.routes.ts:5–10) como padrão de laboratório; rawPayload é a
  união PropertyDataCacheWorkerPayload | FloodEnvironmentalRiskPayload
  (legal-analysis-result.model.ts:57); guard getFloodEnvironmentalRiskPayload
  (legal-analysis-worker-payload.ts:9–12); composição floodRiskCard em
  pages/listing-detail/components/legal-analysis-result-dialog (.ts:122; bloco @if em
  .html:64; Summary em .html:71). A tabela do Card 1 já usa <th scope="row">
  (property-information, .html:66): padrão para a regra ESLint table-scope.
- SUMMARY_CARD_MOCK tem as seções 'Title & Ownership', 'Liens' e 'Public Records' (mock; não
  é alterado).
- Imagem de referência do Card 10: AUSENTE em legal-share (recebida na conversa) -> gate 15.7.

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE E APLICABILIDADE [ESPECÍFICO DO CARD 10, SEGUINDO 3.3]
---------------------------------------------------------------------------------------------
- Resolução própria do Card 10, pelos mecanismos obrigatórios de 3.3: PropertyLookup +
  IPropertyDataCacheReader; com FetchMissingSources=true, MLS ausente buscado SOMENTE por
  SharedMlsQueries (DetailAsync/ListingAsync), compartilhando a consulta com os demais workers.
- O registro MLS é só pré-requisito no Card 10: nada dele é exibido.
- Campos lidos: address.stateOrProvince, address.countyOrParish e public.apn. Coordenadas,
  endereço, tipo e proprietário NÃO são necessários. Não usar FloodPropertyLocationResolver.
- Folio e condado com UM helper interno compartilhado com o Card 8 (ex.: MiamiDadeParcel:
  TryNormalizeFolio -> só dígitos, exatamente 13; IsMiamiDade -> mesmas grafias de
  MiamiDadeEvacuationZoneClient). Motivo: a chave do coordenador precisa ser IDÊNTICA nos
  dois workers para a consulta ser única; duas normalizações divergentes gerariam duas
  cobranças. Quem for implementado primeiro cria o helper; o outro só o usa.
- Aplicabilidade (pré-requisito, 4.1):
  * FL + Miami-Dade + folio válido -> consulta;
  * FL + outro condado -> coverageMissing ("countyClerkNotIntegrated");
  * condado ausente -> coverageMissing ("countyUnconfirmed");
  * fora da FL -> notApplicable;
  * folio ausente ou inválido -> unavailable ("folioUnavailable").
- Conferência de identidade na resposta: cada linha só é aceita se FOLIO_NUMBER (só dígitos)
  for igual ao folio consultado; linhas de outro folio são descartadas com a limitação
  "folioMismatchDiscarded".

---------------------------------------------------------------------------------------------
## 9. FONTES DA FASE 1 [ESPECÍFICO DO CARD 10]
---------------------------------------------------------------------------------------------
### 9.1 Cesta MLS — pré-requisito, fonte COMPARTILHADA
- Leitura pela cesta, sem alterar nenhum DTO; busca quando ausente só via SharedMlsQueries
  (3.2). Proibido chamar IOnlinePropertyDetail direto.
### 9.2 Miami-Dade Clerk — Official Records por folio — conteúdo, fonte COMPARTILHADA (Cards 8 e 10)
- Documentação lida em 2026-11-10: www2.miamidadeclerk.gov/Developers e /Developers/Help
  (rota api/OfficialRecords) e o layout oficial dos arquivos de Official Records
  (miamidadeclerk.gov/library/FTP_File_Layouts/FTP_Layout_Official_Records.pdf), que descreve
  os campos que a API devolve.
- Requisição: GET www2.miamidadeclerk.gov/Developers/api/OfficialRecords
  ?parameter1={folio}&parameter2=FN&authKey={key}. Envelope: Status, StatusDesc,
  UnitsBalance, IPAddress.
- Campos usados pelo Card 10: CFN_YEAR, CFN_SEQ, DOC_TYPE, REC_DATE, REC_BOOK, REC_PAGE,
  BOOK_TYPE, FIRST_PARTY, SECOND_PARTY, PARTY_CODE (D = direct, R = reverse), FIRM_INDIV,
  PARTY_SEQUENCE, ORIG_CFN_YEAR, ORIG_CFN_SEQ, ORIG_REC_BOOK, ORIG_REC_PAGE, LINK_DOCTYPE,
  FOLIO_NUMBER. Campos NÃO lidos: valores financeiros, LEGAL_DESCRIPTION, ORIG_MISC_REF,
  CASE_NUM (Card 8), RecordImages.
- Layout oficial, conferido em 2026-11-10:
  * CFN = ano + "R" + sequência ("02R000001"; ano gravado com 4 dígitos);
  * DOC_TYPE = código do condado ("MOR"); a descrição ("MORTGAGE") existe no arquivo, mas não
    aparece na amostra da API -> tabela fixa de códigos (gate 15.1);
  * partes no formato novo: todas em "First Party", com First Party Code D/R e Cross Party
    Name; exemplo oficial da hipoteca: devedor D (contraparte o banco), banco R (contraparte
    o devedor);
  * Original CFN/Book/Page: referência ao documento original, ex.: a quitação de hipoteca
    citando a hipoteca;
  * Folio Number: número do Property Appraiser;
  * Transaction Type I/U/D e Modified Date: sincronização de arquivos FTP (não usados).
- Compartilhamento (regra 3.2, fonte usada por 2 workers):
  * chave LegalAnalysisSourceKeys.MiamiDadeOfficialRecords(folio), rótulo de log
    "miami-dade-official-records", folio normalizado pelo helper de 8;
  * o cliente devolve TODAS as linhas do folio; cada worker filtra o seu escopo;
  * o contrato já separa falha (exceção) de resultado válido, inclusive "sem registros";
  * o coordenador evita duas consultas SIMULTÂNEAS; não evita uma segunda consulta se a
    primeira já terminou (3.2, "NÃO GARANTE"). Como cada requisição custa US$ 0,20, o
    cliente faz cache-aside em IMemoryCache só de resultados válidos, por folio exato e
    validade curta (MiamiDadeClerkOptions.OfficialRecordsCacheLifetime, 15 min), no padrão
    dos demais clientes de integração (3.1). Falha nunca é cacheada. Decisão 15.2.
  * O Card 8 especificou essa fonte como dele (CARD8_FORECLOSURE.md, 9.3 e 11): quem for
    implementado primeiro cria cliente, chave e cache; o outro só os usa. O blueprint do
    Card 8 precisa registrar a fonte como compartilhada (15.2).
- Custo: 1 unidade por requisição; com o compartilhamento, Cards 8 e 10 juntos fazem 1
  consulta de Official Records por folio na execução (o Card 8 continua com sua consulta
  Civil). O Card 10 não faz nenhuma outra chamada.
- Limites: resposta até 2 MB e até 2.000 linhas; acima disso -> error "responseTooLarge"
  (afeta os dois cards; medir volume real no gate 15.1). Instrumentos no payload: até 50
  (os mais recentes); exibidos: até MaxDisplayedRecords (25), os mais recentes, em ordem
  crescente; a limitação "recordsTruncated" informa o total.
- Segredo: AuthKey na query string por desenho da API; mesmas regras do Card 8 (options,
  nunca em log, exceção, payload ou telemetria; exceção sem URL).
- Estado da fonte: found = ao menos 1 instrumento no escopo; noResults = resposta válida sem
  instrumento no escopo (inclusive com linhas só de outros tipos); unavailable = não
  configurada, saldo zerado, erro ou timeout; error = resposta inválida ou grande demais.
### 9.3 Florida Statutes — NÃO exibidos pelo Card 10
- O CSV cita F.S. §§689.01, 695.01, 28.222, 697.02 e 701.04. A imagem não tem bloco de base
  legal: o card não exibe estatutos e não cria entradas no registro único (3.4) na fase 1.

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD (worker "ownership-public-records", schemaVersion 1) [ESPECÍFICO]
---------------------------------------------------------------------------------------------
{
  schemaVersion, collectedAtUtc,
  jurisdiction: { status, state?, clerk?:"miami-dade",
                  clerkCoverage:"integrated"|"notIntegrated"|"unconfirmed"|"notApplicable" },
  records: { status, sourceId:"miami-dade-clerk-official-records", searchKind:"folio",
             retrievedAtUtc?, totalFound, displayedCount,
             items[]: { family:"deed"|"mortgage"|"assignment"|"satisfactionRelease",
                        docTypeCode, docTypeLabel,
                        recordedDate,
                        cfn?:{ year, sequence }, book?, page?, bookType?,
                        parties: { direct[]:{ display, kind:"firm"|"role" },
                                   reverse[]:{ display, kind:"firm"|"role" },
                                   moreDirect, moreReverse },
                        related?: { cfn?:{ year, sequence }, book?, page?,
                                    linkDocTypeCode?, matchesItemIndex? } } },
  sourceStatus[]: formato padrão (4.1),
  limitations[]
}
- display com kind "role" guarda só o rótulo do papel ("Seller", "Lender"...); o nome de
  pessoa física não existe no payload.
- docTypeLabel vem da tabela fixa código -> rótulo (backend); código sem rótulo conhecido
  -> instrumento fora do escopo (não entra), com a limitação "unknownDocTypeSkipped".
- recordedDate é data de calendário (sem hora, sem fuso).
- Status do worker (regra 4.1): fonte de conteúdo = Official Records. found ->
  dataAvailable; noResults -> noResults; senão -> sourceUnavailable. Cesta MLS, folio e
  jurisdição são pré-requisito e não geram dataAvailable.
- Limitações (códigos fixos): folioIndexedSearchOnly (sempre que houve consulta: a busca só
  acha instrumentos indexados com o folio), relatedRecordNotProofOfDischarge (sempre que
  houver related), recordsTruncated, folioMismatchDiscarded, unknownDocTypeSkipped,
  countyClerkNotIntegrated, countyUnconfirmed, folioUnavailable, payloadReduced.
- Sem payload bruto, sem folio, sem endereço, sem valores. Payload alvo <= 20 KB
  (MaxPayloadBytes 40.000); ao estourar, reduzir primeiro os itens mais antigos, com
  "payloadReduced".
- Projeções (classe estática OwnershipPublicRecordsProjection):
  * Read: valida schemaVersion, enums, datas e coleções antes de qualquer uso;
  * ForApi: payload público do card;
  * ForReport: resumo para o LLM — clerkCoverage, totalFound, contagem por família, data de
    registro da escritura mais recente, quantidade de instrumentos com registro relacionado,
    sourceStatus, limitations. Sem nomes (nem de empresa), sem CFN, sem texto livre, sem
    inferência de hipoteca aberta ou quitada.

---------------------------------------------------------------------------------------------
## 11. BACKEND [ESPECÍFICO DO CARD 10, SEGUINDO 3, 4 E 5]
---------------------------------------------------------------------------------------------
- Application/Contracts/OwnershipPublicRecordsContracts.cs: payload tipado. Sem porta nova: o
  Card 10 usa IMiamiDadeClerkRecordsClient.GetOfficialRecordsByFolioAsync (especificada no
  Card 8; criada por quem for implementado primeiro, com DTO de linha que contenha os campos
  de 9.2 dos dois cards).
- Application/Contracts/OwnershipPublicRecordsProjection.cs: Read, ForApi, ForReport.
- Application/Options/OwnershipPublicRecordsOptions.cs: { SourceTimeout=8 s,
  TotalBudget=15 s, MaxPayloadBytes=40.000, MaxDisplayedRecords=25 }; validação no
  NewLegalAnalysisModuleDependencyInjection (SourceTimeout <= TotalBudget; TotalBudget <
  WorkerTimeout; MaxPayloadBytes < limite geral; MaxDisplayedRecords entre 1 e 50),
  conforme 4.1.
- Infrastructure/Orchestration/LegalAnalysisSourceKeys.cs: acrescentar
  MiamiDadeOfficialRecordsLabel e MiamiDadeOfficialRecords(folio) (método por fonte, 3.2).
- Tabela fixa de documentos (código -> família e rótulo) e de papéis (família + lado ->
  rótulo): classe estática interna, versionada, sem banco e sem DI, montada com a lista
  oficial do Clerk (gate 15.1). A mesma tabela serve ao Card 8 para os códigos de lis
  pendens: UMA tabela, não uma por card.
- Integrations: cache-aside no cliente do Clerk (9.2), se ainda não existir; nenhuma outra
  alteração.
- Infrastructure/Workers/OwnershipPublicRecordsWorker.cs:
  Id = LegalAnalysisWorkerIds.OwnershipPublicRecords ("ownership-public-records"); registro
  explícito em AddNewLegalAnalysisWorkers. Sequência: identidade (seção 8) -> consulta
  context.QuerySourceAsync(chave, token => clerk.GetOfficialRecordsByFolioAsync(...),
  waitToken), isolada com timeout -> conferência de folio -> agrupamento por instrumento ->
  filtro de escopo -> partes (D/R, empresa x papel) -> registros relacionados -> ordenação e
  limite -> payload. Status pela regra 4.1 (seção 10). TimeProvider; ILogger +
  ILegalAnalysisRunTrace com RunId, sourceId, status, durationMs, totalFound e
  unitsBalance; nunca folio, CFN, nome de parte, AuthKey, URL ou payload.
- LegalAnalysisReportPromptBuilder: switch por WorkerId (seção 5, item 5) com
  OwnershipPublicRecords -> OwnershipPublicRecordsProjection.ForReport; demais inalterados.
  O LLM nunca conclui sobre título, quitação ou gravame.
- LegalAnalysisWorkerPayloadMapper: acrescentar OwnershipPublicRecords ->
  OwnershipPublicRecordsProjection.ForApi, mantendo os casos existentes.
- Não alterar: coordenador, orquestrador, cesta, clientes MLS/RentCast/Parcel, DTOs MLS,
  persistência, contratos HTTP, outros workers.

---------------------------------------------------------------------------------------------
## 12. FRONTEND (Angular) [ESPECÍFICO DO CARD 10, SEGUINDO 4.2 E 5]
---------------------------------------------------------------------------------------------
- data-access/models/legal-analysis-ownership-payload.model.ts (payload) e
  legal-analysis-ownership.model.ts (view model).
- legal-analysis-result.model.ts: OwnershipPublicRecordsPayload na união de rawPayload.
- legal-analysis-worker-payload.ts: getOwnershipPublicRecordsPayload (workerId
  'ownership-public-records' + schemaVersion 1), no padrão de
  getFloodEnvironmentalRiskPayload.
- data-access/legal-analysis-ownership.mapper.ts + .spec + .fixture: payload -> view model
  (linhas da tabela, "→" entre os lados, "+<n>", datas "MMM d, y" en-US, referência do
  Clerk, linhas de registro relacionado, estado vazio x indisponível, rodapé).
- Formatador de CFN ÚNICO para os Cards 8 e 10 (data-access, ex.: legal-analysis-clerk-
  reference.ts), com o formato confirmado no gate 15.1. Quem for implementado primeiro cria.
- ui/legal-analysis/cards/ownership-public-records/: standalone, signals, colapsável (seção
  5, item 11).
- Layout expandido — a imagem de referência manda (PROMPT, Regra 7):
  1) Cabeçalho: tile com ícone de documento + título em gradiente + subtítulo "Official
     record index" abaixo do título + barra decorativa em gradiente à direita do título +
     chip "Illustrative data" (só laboratório, input sampleData) + seta.
  2) Tabela em painel com cabeçalho em faixa azul-clara: colunas Instrument (~27%), Recorded
     (~15%), Parties (~33%), Clerk reference (~25%); divisórias finas entre linhas.
  3) Linha de registro relacionado em texto pequeno: rótulo em itálico + aviso fixo.
  4) Divisória + rodapé "Source: Miami-Dade Clerk Official Records API" (fonte em negrito
     no rótulo "Source:").
- Estado vazio e estado indisponível: tile com ícone de documento e lupa, título em negrito,
  linha secundária, divisória e rodapé (só no vazio), como na imagem.
- Tabela acessível: <table> com <caption> visualmente oculto ("Official record index"),
  <th scope="col"> nas colunas; seta "→" com texto alternativo ("to") para leitor de tela.
- Linha compacta (está na imagem): título + divisória vertical + "<n> indexed records".
- Responsividade: até 760 px a tabela vira lista de cartões (rótulo · valor por linha,
  mantendo a ordem das colunas); até 600 px o subtítulo vai para baixo da barra decorativa.
- Rota de teste: features/listings/pages/card-ownership (path "card-ownership", fora do
  Shell, canMatch !environment.production, loadComponent), no padrão de pages/card-flood,
  com mocks dos três estados da imagem e do estado indisponível.
- Composição no diálogo: computed ownershipCard + bloco @if após o bloco do card de número
  anterior já implementado (Card 9, 8, 7, 6 ou 5) e antes do Summary.
- Regras ESLint de template do AGENTS.md (table-scope, valid-aria, elements-content). Sem
  link para imagem de documento e sem PDF na fase 1.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE
---------------------------------------------------------------------------------------------
Backend (unit): agrupamento por instrumento (uma linha por parte); lados D/R; empresa x
pessoa física (FIRM_INDIV) e marcação ausente -> papel; papéis por família; mais de 2 partes
-> "+n"; lado vazio; filtro de escopo (lis pendens, liens, NOC e códigos desconhecidos fora);
FOLIO_NUMBER divergente descartado; ordenação crescente com desempate por CFN; truncamento
nos 50/25 mais recentes; registro relacionado por CFN, por book/page, coincidente com linha
da tabela e ausente; REC_DATE como data; Miami-Dade / outro condado / condado ausente / fora
da FL / folio ausente ou inválido; resposta vazia (noResults), só com outros tipos
(noResults), grande demais, Status de erro, saldo zerado, timeout; orçamento com
TimeProvider fake; limite de payload; projeções Read/ForApi/ForReport; payload mapper;
prompt sem nomes, CFN ou texto livre.
Backend (compartilhamento): Cards 8 e 10 na mesma execução fazem UMA requisição de Official
Records (coordenador em paralelo; cache-aside quando um termina antes); chaves idênticas
pelo mesmo helper de folio; falha não é cacheada; cada worker filtra o próprio escopo da
mesma resposta.
Backend (segurança): AuthKey ausente de log, exceção, payload e telemetria; nome de pessoa
física ausente de payload, log e prompt.
Backend (reaproveitamento): MLS buscado por SharedMlsQueries (com e sem coordenador no
contexto, como em LegalAnalysisSourceCoordinatorTests); nenhuma chamada direta a
IOnlinePropertyDetail no worker; leitura pela cesta antes de buscar; nenhum uso de
FloodPropertyLocationResolver; uma única tabela de códigos de documento.
Backend (integração): fixtures gravadas do gate 15.1 (folio com escritura, hipoteca e
quitação com referência ao original; folio sem registros; folio com lis pendens, para o
filtro); worker no orquestrador sem afetar property-data-cache e Flood; WebApiBoundaryTests
e suíte completa.
Frontend: specs do mapper (estados com registros, vazio e indisponível; formatador de CFN
compartilhado; "→" e "+n"; linha de registro relacionado), do componente
(expandido/compacto/tabela responsiva) e dos guards; nenhum texto "Clear title", "Paid off"
ou "Open mortgage" renderizado; smoke desktop/mobile com capturas no work item, comparadas
com a imagem de referência.
Homologação real: imóvel em Miami-Dade com escritura, hipoteca e quitação; imóvel em
Miami-Dade sem registro indexado por folio; imóvel fora de Miami-Dade; Clerk indisponível;
execução completa com Card 8 ativo (uma requisição só). Conferir no log/LegalAnalysis.txt os
eventos Worker e SourceQuery (Started/Joined) e o saldo de unidades.
Aceite: todos os campos da imagem dinâmicos e com fonte; cards já implementados e Summary
inalterados; nenhum dado de outro card repetido; nenhuma conclusão sobre título ou quitação;
nenhum nome de pessoa física exibido; uma requisição de Official Records por execução;
execução total < 120 s.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO
---------------------------------------------------------------------------------------------
1) Validação de dados e fontes: este documento + conferência anti-duplicidade + decisões
   15.2 a 15.5 + gate 15.1 (conta de Developer do Clerk, já exigida pelo Card 8: iniciar
   já).
2) Rota card-ownership com HTML estático nos três estados da imagem, no estado indisponível
   e na versão compacta; aprovação visual.
3) Peças compartilhadas com o Card 8, se ele ainda não as criou: cliente do Clerk (Official
   Records) com cache-aside, chave no LegalAnalysisSourceKeys, helper de folio/condado,
   tabela de códigos de documento, formatador de CFN.
4) Worker, options, registro no DI, projeções, PromptBuilder, PayloadMapper,
   modelo/guard/mapper/componente Angular, composição no diálogo.
5) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile, execução real (com o Card 8,
   se já existir) e leitura do log.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
---------------------------------------------------------------------------------------------
15.1 Miami-Dade Clerk (mesma conta e mesmo AuthKey do gate 15.2 do Card 8): com chamadas reais
     a api/OfficialRecords por folio, anotar: nomes dos campos no JSON (a documentação só
     mostra o corpo completo em XML); lista oficial de códigos DOC_TYPE com descrição
     (deeds, mortgage, assignment, modification, satisfaction/release, lis pendens, liens,
     judgments), inclusive se existe código próprio para warranty deed; valores reais de
     PARTY_CODE (D/R) e de FIRM_INDIV; lado D/R real em deed, mortgage, assignment e
     satisfaction; formato oficial de exibição do CFN (zeros à esquerda); volume de linhas
     por folio; cobertura da indexação por folio (quais documentos não trazem
     FOLIO_NUMBER). Bloqueia os passos 3 e 4.
15.2 Fonte compartilhada com o Card 8: aprovar o cache-aside no cliente do Clerk (só
     resultado válido, por folio exato, 15 min) e atualizar o blueprint do Card 8 (9.3 e
     11) para registrar Official Records como fonte compartilhada, com a chave
     MiamiDadeOfficialRecords e o helper de folio. Sem o cache, uma execução pode pagar a
     consulta duas vezes (o coordenador só une consultas simultâneas).
15.3 Privacidade das partes (interpretação da imagem): nomes só de empresas; pessoas físicas
     pelo papel ("Seller", "Borrower"...). Recomendação: aprovar, porque reproduz a imagem e
     mantém nomes de pessoas fora do payload, do Blob, do LLM e do log. Alternativa: exibir
     todos os nomes como o índice público publica, com aprovação de produto/jurídico e
     revisão do armazenamento no Blob (mesma questão do Card 8, 15.4).
15.4 Rótulo do instrumento: o card mostra o rótulo do código do Clerk. Se o índice não
     distinguir "Warranty Deed", a linha mostra "Deed" e a imagem diverge nesse texto.
     Alternativa: subtipo pela RealEstateAPI PropertyDetail (saleHistory[].documentType),
     que acrescentaria fonte, custo e casamento por data — não recomendado.
15.5 Escopo: aprovar a fase 1b (liens, claims of lien, judgments e liberações na mesma
     tabela, sem chamada nova), com o texto do estado vazio "No indexed records found in this
     search". Recomendação: aprovar logo após a fase 1, porque o CSV ("Liens e gravames
     registrados") e o blueprint do Card 6 atribuem esses dados ao Card 10.
15.6 Uso de LLM no card: em aberto (arquiteto Bruno, PROMPT 10E). O worker entrega fatos; o
     LLM só recebe ForReport.
15.7 Imagem de referência: salvar em legal-share (padrão CARD10_OWNERSHIP_IMG_REF.<ext>);
     aprovar o estado indisponível e as linhas compactas propostas (6.1); confirmar que
     "· when supplied" é anotação de design.
15.8 Base de código (reconferido em 2026-11-10): d_leevli-platform-api está em
     feat/legal-analysis-source-coordinator (57074dd6 "Modulariza Workers"), que contém o
     Card 5 e o Plano A; d_leevli-platform-web está em LEGAL (71e79913). Os Cards 6 a 9 não
     estão implementados: peças compartilhadas (switch do PromptBuilder, helper de MLS de
     3.3, cliente e chave do Clerk, helper de folio, tabela de códigos, formatador de CFN)
     são criadas por quem for implementado primeiro; os outros só acrescentam.
15.9 Work item com critérios de aceite; branch feat/<card>-ownership-public-records;
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
