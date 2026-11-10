# LEEVLI | LEGAL ANALYSIS | CARD 12 — PERMITS AND CONSTRUCTION COMPLIANCE
# MASTER IMPLEMENTATION BLUEPRINT V1 — TEMPLATE DO CARD 6

Blueprint de planejamento, não implementação. Referências: LISTA_CARDS.txt; PROMPT_DEV_LEGAL_CARDS.txt; Fontes_API_Legal_analyses.csv; CARD12 PERMITs.png; blueprints dos Cards 6–11, 13 e 14. Conferência documental em 10/11/2026. A imagem existente é a referência visual; não é dado real de um imóvel. As seções [REGRA FIXA] abaixo reproduzem o texto comum dos blueprints existentes. Suas referências de arquivos e datas são históricas e devem ser reconferidas antes da implementação; os arquivos de blueprint foram renomeados durante esta análise. Em caso de conflito, prevalecem as instruções do projeto, o código atual e os gates deste documento. Este blueprint não comprova integração nem cobertura completa das fontes.

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
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 12]

Exibir registros de licenças de construção associados de forma comprovável ao imóvel, com o detalhe do registro selecionado. A imagem define duas versões de acordeão, expandida e compacta, e um estado vazio. O escopo do CSV inclui também histórico, permits abertos/expirados, code enforcement, CO e possíveis divergências de obras; a imagem não contém todos esses itens. A fase 1 não declara esses itens ausentes ou verificados quando a fonte não os cobre. Não acrescentar novos painéis à imagem sem decisão de produto.

### 6.1 Contrato visual e mapeamento de cada dado

Fonte candidata principal: camada oficial Miami-Dade County Building Permits, ArcGIS MapServer/1 [S1]. A documentação lista atributos e tipos, mas uma resposta de dados da operação query não pôde ser acessada nesta pesquisa. "Campo documentado" não significa que haverá valor para todos os imóveis ou licença para uso em produção. O cliente e a integração ainda não existem. Não usar os exemplos da imagem como dados de produção.

| Elemento da imagem | Mapeamento proposto | Regras e limites |
|---|---|---|
| Permit records (3 shown) / 3 shown | Contagem da lista efetivamente exibida | Não confundir com todos os registros existentes; o número 3 da imagem é ilustrativo. Contagens e rótulo devem indicar se há truncamento. |
| Permit number | ID (número do permit) | API o declara double; preservar representação inteira segura, não interpolar ponto decimal. Garantir identidade estável e unicidade no gate 15.1. |
| Category | DESC1...DESC10, com CAT1...CAT10 | A fonte descreve múltiplas categorias; definir no gate 15.3 representação sem descartar ou inventar rótulos. Não confundir com TYPE. |
| Status | BPSTATUS | Códigos documentados A=Active, E=Expired, F=Finalized; códigos desconhecidos/ausentes são Unknown/Not provided, nunca classificados por inferência. |
| Issued date | ISSUDATE | Campo ArcGIS date; validar unidade epoch e fuso na resposta real. Data civil sem deslocamento incorreto. |
| Last inspection | LSTINSDT | String até 8 caracteres; formato, sentinelas e fuso não estão confirmados. Não chamar de inspeção final/aprovada. |
| Radio e destaque da linha | Identificador da linha selecionada | Estado de interface, sem fonte externa; detalhes sempre do registro selecionado. |
| Selected permit details — número (status) | ID + BPSTATUS do registro selecionado | Se número/status indisponível, não fabricar título; usar alternativa clara. |
| Process | PROCNUM | String; preservar zeros e caracteres. Não confundir número do processo com ID do permit. |
| Type | TYPE | Mostrar o valor fornecido; não traduzir ou presumir que ROOF é descrição oficial. |
| Master permit | MPRMTNUM | Número double; validar formato, zeros/sentinelas e vínculo. Não pressupor que todos os subsidiários foram encontrados. |
| Last approval | LSTAPPRDT | String de data; não equiparar aprovação à conclusão de obra. |
| Renewed | RENDATE | String de data da renovação; não inferir status atual. |
| CO/CC date | BLDCMPDT | Data de CO/CC indicada pela fonte, sem inferir qual tipo, existência de documento, validade ou autorização atual. |
| Contractor | CONTRNAME | Nome da fonte; não comprova habilitação ou contratação atual. |
| Contractor no. | CONTRNUM | Número informado; não presumir licença válida ou formatar removendo zeros. |
| Folio | FOLIO | String de 13 posições; apresentar valor do registro da licença, não substituir pelo folio consultado. Gate 15.4 de possível duplicidade. |
| Address | ADDRESS | Endereço do registro do permit; não substituir pelo endereço do anúncio. Gate 15.4 de possível duplicidade. |
| Unit | UNIT | Unidade do permit; vazio não comprova licença para todo o edifício nem para unidade. |
| Last inspection no compacto | Máximo de LSTINSDT válido dentre os registros mostrados | Não é necessariamente a inspeção mais recente de todo o imóvel; ausência é Not provided, falha é Not available. O conjunto deve coincidir com o dos contadores. |
| Chips 1 active / 1 expired / 1 finalized | Contagem dos BPSTATUS nos registros mostrados | Informar categoria Unknown quando houver códigos não mapeados; nunca ocultar valores desconhecidos para produzir totais aparentes. |
| Illustrative data, Variation 1, rodapé Illustrative records... | Identificação explícita de mock e sua cobertura no laboratório | Não são fatos de propriedade; nunca mostrar como dados reais no card definitivo. Produção precisa de identificação da fonte e limitação temporal/jurisdicional. |
| No permits found / No permits were found in the records searched | Estado visual vazio | Texto pode ser usado no laboratório. Em produção, dizer "No records returned by the sources searched" e indicar escopo, período e possíveis lacunas. Erro ou jurisdição desconhecida não é resultado vazio. |
| Chevron | Estado local do acordeão | Inicia expandido; alterna via botão acessível. No compacto mostrar título, quantidade exibida, chips e última inspeção como na imagem. |

Toda data de string só é convertida após o gate 15.1 confirmar formato e valores especiais. Nulo, vazio, valor inválido e fonte indisponível não são equivalentes: campo ausente em resposta válida = Not provided; consulta não executada, falha ou valor não interpretável = Not available. Datas inválidas mantêm limitação, nunca se tornam data padrão. Para apresentação usar datas locais de calendário verificadas, sem criar horário artificial.

### 6.2 Semântica e limites

- A fonte [S1] é do condado, não o conjunto de todos os emissores municipais. Metadados oficiais associados descrevem janela de três anos e atualização semanal [S2]; a equivalência da documentação com a camada atual, janela efetiva, último refresh, exclusões e termos precisam ser confirmados. Tratar como cobertura limitada; não prometer histórico completo ou atualização em tempo real.
- O condado orienta que municípios incorporados têm seus próprios Building Departments [S3]. Permits do condado podem existir para imóveis municipais, mas não substituem a consulta municipal. Não presumir que uma fonte é completa porque retorna dados. Folio prefixado 30 pode ajudar na identificação da área não incorporada, porém competência e cobertura concretas exigem confirmação; município/condado não devem ser inferidos por aproximação frágil.
- A API documenta FOLIO, ADDRESS e UNIT como atributos do permit; dados de condomínio podem estar vinculados ao folio mestre ou unidade, sem vínculo individual inequívoco. Não mesclar endereços, vizinhos, folios mestres ou unidades por proximidade, nem buscar todos os imóveis de um prédio e atribuí-los à unidade. Match de folio não prova associação de todos os registros à unidade quando outros sinais conflitam; sinalizar ambiguidade.
- "Finalized" é o status fornecido para um permit; não prova conclusão da obra inteira, conformidade, autorização de ocupação ou ausência de outro permit aberto. Um zero na lista consultada não certifica a inexistência de obras sem licença, violações ou passivos.
- O CSV distingue CO, CC e CU: BLDCMPDT é rotulado CO/CC; uma data só não identifica o certificado nem comprova sua validade. Pesquisa oficial de certificados e histórico pode existir como portal [S4], mas não há API de certificados integrada ou programaticamente validada neste trabalho.
- Violações, holds, requisitos, inspeção final, expiration date, histórico integral e supostas obras não autorizadas não estão mapeados nos atributos conferidos. Não criar valor, conclusão ou painel para eles. Registrar cobertura parcial; novas fontes ou comparação de áreas demandam especificação e validação próprias. Divergência de área nunca prova obra ilegal.

### 6.3 Antiduplicidade

- Card 1 é dono da identidade do imóvel; Folio/Address/Unit na imagem são valores específicos do registro de permit e podem coincidir com os dados do Card 1. Isso conflita potencialmente com a regra fixa de não repetir dados; gate 15.4 decide como preservar o contrato visual sem duplicação indevida. Não alterar unilateralmente a imagem.
- Card 7 é dono da aprovação privada da associação e referencia o permit público do Card 12; não misturar as exigências.
- Card 9 trata segurança estrutural; referência anterior atribui ao Card 12 permits e idade de telhado por permit, mas a imagem não contém idade e a fonte não comprova idade: fora desta fase.
- Card 10 trata instrumentos e registros de titularidade; um permit não é instrumento de propriedade nem lien. Card 11 é dono de impostos; Card 13 de uso/zoneamento público; Card 14 de aluguel/investimento. Não repetir seus dados. Code-enforcement pode ter gravame, mas caso aberto não comprova lien; verificar vínculo e titularidade antes de qualquer extensão.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO [ESPECÍFICO DO CARD 12]

- Em 10/11/2026, backend na branch feat/legal-analysis-source-coordinator e frontend na branch LEGAL. O DI registra property-data-cache e flood-environmental-risk; não foi encontrado worker/cliente de permits no NewLegalAnalysis.
- Existe SourceCoordinator por execução com QuerySourceAsync, execução concorrente de workers e contexto independente de Buildings.Id. IPropertyDataCacheReader é somente leitura; a política de preencher faltas pertence ao fluxo existente. Um worker não pode depender da ordem ou do resultado de outro.
- A API usa LegalAnalysisWorkerPayloadMapper: reconhece property-data-cache e Flood; IDs desconhecidos retornam null. LegalAnalysisReportPromptBuilder projeta Flood de forma específica. Novo payload requer projeções explícitas e ajuste conjunto de contratos, mapper da API, prompt e consumidor Angular. A integração não surge ao registrar o worker.
- O log local existe; a última execução registrada em 06/11/2026 concluiu com workers property-data-cache e flood-environmental-risk. Isto não confirma funcionamento do Card 12 nem disponibilidade da fonte externa.
- Blueprints dos outros cards são planos, não prova de implantação. Os textos fixos são referência datada; confirmar código, AGENTS e configurações na branch efetiva antes de implementar. Sem mudança nesta análise em código, dados, schema ou ambientes.

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE E APLICABILIDADE [ESPECÍFICO DO CARD 12]

1. Obter estado, condado, identidade do parcel e unidade de fontes já disponíveis pelo fluxo existente e snapshot confiável; antes de criar resolvedor, avaliar PropertyLookup, IPropertyDataCacheReader e IPropertyParcelClient. Usar fonte e chaves existentes somente após confirmação de equivalência; não acoplar ao FloodPropertyLocationResolver específico nem depender de outro worker.
2. Somente consultar [S1] se jurisdição e identidade forem suficientes. Validar que o folio de entrada é string de exatamente 13 dígitos após remover apenas separadores reconhecidos; preservar zeros à esquerda. Não converter números que já perderam zeros; não aceitar concatenação ou caracteres que possam alterar a expressão ArcGIS. Se APNs de fontes confiáveis divergirem ou condado não for comprovado, devolver identidade ambígua/indisponível e não associar resultados.
3. Comparar FOLIO retornado ao folio consultado e examinar informações conflitantes de unidade e localização. A normalização do endereço é apenas conferência auxiliar, não prova isolada. Se UNIT for ambígua ou o escopo prédio/unidade for desconhecido, não apresentar registro como licença confirmada da unidade; comunicar limitação. Permits de áreas comuns não devem desaparecer silenciosamente: a associação segura requer regra aprovada e dados de fonte apropriados.
4. Fora da jurisdição coberta, condado desconhecido ou falta de folio: indicar fonte não aplicável ou não verificada conforme evidência; não emitir "No permits found". Não extrapolar para todo o estado ou condado. Não usar endereço aproximado, nome de proprietário, dados inventados nem Buildings.Id.

---------------------------------------------------------------------------------------------
## 9. FONTES, CONSULTA E PROVENIÊNCIA [ESPECÍFICO DO CARD 12]

[S1] Esquema e query da camada County Building Permits @ BuildingPermit: https://gisweb.miamidade.gov/arcgis/rest/services/MD_LandInformation/MapServer/1 . A camada anuncia JSON, campos listados na seção 6.1, query, paginação, maxRecordCount 1000 e ordenação. A operação query existe, mas tentativa de consulta de dados pela ferramenta de pesquisa não ficou acessível; não afirmar que resultado JSON, paginação efetiva, formato das datas ou valores estão homologados.
[S2] Metadados oficiais do conjunto BuildingPermit: https://www.arcgis.com/sharing/rest/content/items/31cd319f45544648b59f0418aea60091/info/metadata/metadata.xml?format=default&output=html . Descrevem três anos e atualização semanal; verificar vínculo e validade atuais para [S1] antes de especificar o período.
[S3] Competência e municípios: https://www.miamidade.gov/global/economy/building/homeowners-general-permit-info.page .
[S4] Consultas oficiais de permits, certificados e code compliance: https://www.miamidade.gov/global/service.page?Mduid_service=ser149695713973833 e https://wwwx.miamidade.gov/Apps/RER/ePermittingMenu/Home/Permits . Portal consultável por humano não comprova API estável, autorização de scraping ou completude.
[S5] Paginação ArcGIS: https://developers.arcgis.com/rest/services-reference/enterprise/query-map-service-layer/ . Respeitar exceededTransferLimit e estabilidade da ordenação; página com menos registros, inclusive vazia, não prova fim enquanto houver sinal de mais dados.
[S6] CSV e PROMPT locais: as cinco linhas listadas na seção 6, com F.S. §553.79, cap. 162 e FBC conforme aplicabilidade. São referências de escopo, não fundamentação para conclusão jurídica automática; confirmar versão e emendas quando houver implementação dos itens correspondentes.

Consulta futura: HTTPS, host e caminho configurados/fixados e validados; outFields limitado aos atributos necessários, returnGeometry=false, filtro de igualdade pelo folio estritamente validado, resposta limitada e cancelamento. Evitar imprimir URL completa, folio, endereço, nome ou payload nos logs. Tratar erros HTTP e erro ArcGIS dentro de HTTP 200 como falha; validar schema. Fonte externa é dado não confiável, não instrução de LLM.

Paginar com ordem estável documentada (OBJECTID como desempate), respeitar orçamento, limite de páginas/registros e tamanho máximo do worker; não usar loop infinito. Registrar truncamento se exceededTransferLimit persistir, se limite/timeout atingir, ou houver mudança de paginação; não prometer snapshot transacional de fonte mutável. Páginas incompletas não permitem dizer que todas as licenças ou status foram analisados. Sem cache novo ou retry especulativo; coordenador só quando houver consultas efetivamente compartilhadas. Prazo do worker respeita deadline do contexto e cancelamento. Custos, disponibilidade, limites e termos do serviço não foram confirmados.

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD [PROPOSTA, NÃO IMPLEMENTADO]

WorkerId proposto: permits-construction-compliance; schemaVersion 1. Definir tipos C# e TypeScript compatíveis, nulabilidade explícita e enums conhecidos. Estrutura lógica:

- schemaVersion, collectedAtUtc, source: { id, status, queriedAtUtc, coverageDescription, sourceUpdatedAtUtc? }, limitations[];
- identity: { status, matchScope, reason? } sem incluir identidade privada desnecessária na projeção pública;
- result: { completeness, truncated, records[] }, com número de registros retornados/exibidos e desconhecido quando não puder ser calculado. Nenhuma contagem global inventada;
- record: { recordKey, permitNumber?, categoryDescriptions[], status, issuedDate?, lastInspectionDate?, processNumber?, type?, masterPermitNumber?, lastApprovalDate?, renewalDate?, coCcDate?, contractorName?, contractorNumber?, folio?, address?, unit? }, com campos opcionais e estados explícitos onde necessário.

Estados do worker devem usar LegalAnalysisWorkerStatus existente: DataAvailable para registros confiavelmente associados, com limitations se parcial; NoResults apenas após resposta válida, paginação concluída e consulta aplicável, significando nenhum registro na fonte/escopo pesquisados; SourceUnavailable, InvalidResponse, Failed, TimedOut e Cancelled conforme contrato. NoResults nunca garante histórico completo. Identidade ambígua ou falta de cobertura não vira NoResults. source.status e completeness distinguem desconhecido, consulta não realizada, parcial e completo dentro da fonte; não adicionar enum novo ao contrato global sem decisão explícita.

Selecionar linha, contadores e data compacta no frontend sobre o mesmo conjunto exibido; em truncamento, rotular "shown" e informar que pode haver outros. Exibir status desconhecido; contadores não devem sugerir que os três grupos representam todos quando não representam. O pacote privado pode registrar evidência mínima seguindo regras de retenção/privacidade; projeções ForApi e ForReport expõem somente campos necessários. Não enviar folio, endereço, unidade, nome/número de contratado nem payload bruto ao LLM por padrão; relatório agregado recebe fatos e limitações mínimos, nunca conclui conformidade jurídica. Verificar termos antes de persistir payload em workers.json.

---------------------------------------------------------------------------------------------
## 11. BACKEND [PROPOSTA]

- Criar apenas após resolver gates: cliente ArcGIS tipado, registrado por DI e Options validadas conforme padrões, em Infrastructure; contrato de consulta e payload em Application. Não duplicar mecanismos existentes. Nenhum acesso a SQL/Buildings no worker; chamadas HTTP com cancellation, limites e tratamento de falhas isoladas. Fonte e dados não alteram o fluxo de relatório/persistência existente.
- Reaproveitar leitura de identidade, validação e QuerySourceAsync quando adequados; não acionar PropertyParcel por dependência do outro worker e não lançar consulta paga sem validação de necessidade e custo. Revalidar coincidência de identidade.
- Adicionar ID e registro explícito, projeções Read/ForApi/ForReport e os casos no LegalAnalysisWorkerPayloadMapper e no LegalAnalysisReportPromptBuilder; validar limites de serialização e nulidade. Não emitir query, dados pessoais ou payload nos logs; usar ILogger para dados operacionais seguros e TimeProvider para tempo.
- Preservar sequência existente: relatório validado, Building confirmado, tentativa Blob, PropertyReport em commit independente. Falha Blob não impede SQL; falha SQL não exclui Building. Sem migration, tabela, coluna, transação prolongada ou novo arquivo de auditoria.

---------------------------------------------------------------------------------------------
## 12. FRONTEND (ANGULAR) [PROPOSTA]

- Após validar fontes e divergências conforme PROMPT, criar laboratório estático dedicado na rota /card-permits, com dados claramente ilustrativos, sem integrá-lo ao relatório. Informar rota antes da criação. Produzir expandido com tabela selecionável à esquerda e detalhes à direita, vazio e compacto como na imagem; um único acordeão funcional por exemplo, expandido por padrão.
- Cabeçalho: ícone de documento, título em gradiente, separador e "Variation 1" apenas no laboratório, chip Illustrative data e chevron. Tabela: cabeçalhos, status em chips, radios acessíveis e linha selecionada. Detalhe: número/status e duas colunas de atributos com ícones. Rodapé de mock mostra que não há correspondência com imóvel real.
- Compacto: título, número mostrado, contagens de status, última inspeção do conjunto mostrado e chevron; tratamento de ausências, desconhecidos, truncamento e falta de cobertura deve ser revisado com produto sem fingir correspondência completa com a imagem.
- Botão com aria-expanded, nome acessível e suporte a teclado; tabela com th scope adequado, radios com rótulos e foco visível; sem autofocus. Layout responsivo: empilhar painéis e permitir rolagem horizontal da tabela quando necessário sem ocultar conteúdo. Respeitar acessibilidade, estilos e componentes existentes.
- Após aprovação visual e gates: adicionar tipos/guard/mapper do payload na camada data-access, componente standalone reutilizável em ui/legal-analysis/cards e composição no diálogo na posição 12; não mover outros cards. Valores do laboratório nunca entram no componente de produção. Fonte ausente/erro/identidade ambígua têm estado próprio, não tela de zero. Informar jurisdição e janela da fonte em texto legível; não usar "Illustrative data" para dados reais.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE

- Contrato: desserialização dos campos, enum A/E/F e valor desconhecido, ausência/nulo, datas válidas/inválidas/sentinelas, números inteiros e precisão, zeros à esquerda, ID duplicado e múltiplas categorias. Cobrir parsers com fixtures reais anonimizadas após autorização e verificação dos termos, não assumir exemplos como amostras reais.
- Identidade: folio exato, inválido, zeros perdidos, dados conflitantes, jurisdição desconhecida/fora da cobertura, condo com folio mestre/unidade divergente e fonte vazia; nenhum match aproximado indevido.
- Fonte: resposta válida vazia, paginação completa, exceededTransferLimit true até em página vazia, limite de páginas, repetição, timeout, erro ArcGIS em HTTP 200, schema inválido, fonte indisponível, cancelamento, fonte mutável e atualização desconhecida. Garantir que resposta parcial não declara ausência total.
- Integração: registro DI, contratos worker e API, projections, prompt sem dados privados, limites de payload, execução paralela e isolamento, ausência de Buildings.Id, preservação de Building/Blob/PropertyReport, WebApiBoundaryTests e suíte pertinente.
- UI: seleção muda somente os detalhes corretos; três grupos e unknown; contagens e última inspeção pertencem aos registros mostrados; vazio bem-sucedido distinto de erro e parcial; sem valores demonstrativos em produção; teclado, foco e aria, expansão/colapso, desktop e mobile. Capturas de smoke no work item e comparação visual com a imagem.
- Aceite depende dos gates e da aprovação visual; não declarar todos os campos recuperáveis por imóvel, cobertura completa, ausência de violações ou obra regular com base apenas no schema. Testar com imóveis autorizados e fonte confirmada, sem divulgar dados sensíveis em logs.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO

1. Resolver gates de fonte, cobertura, identidade, contratos e duplicidade; validar com dados reais autorizados e documentação/termos. Apresentar e aprovar divergências dos dados antes do HTML, como exige o PROMPT.
2. Criar somente o laboratório /card-permits com expandido, compacto e vazio; aprovar visualmente desktop/mobile. Dados são ilustrativos e sem vínculo com imóvel real.
3. Com gates aprovados, implementar worker/cliente mínimo e projeções, integrar frontend dinâmico, preservar estados de parcialidade e sem dados. Sem fontes extras inferidas.
4. Rodar testes pertinentes, arquitetura e smoke; validar pipeline e persistência existentes. Documentar limitações remanescentes e cobertura real. Code enforcement, CO documental, inspeções detalhadas e comparação de obras dependem de definição e fonte apropriada antes de implementação.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO

15.1 BLOQUEANTE — Exercitar operação query autorizada de [S1], sem divulgar dados de terceiros; verificar JSON efetivo, filtragem por folio, atributos e respectivos valores, datas/sentinelas, ID seguro, codificação e disponibilidade; validar casos reais conhecidos e paginação. Nesta pesquisa a página de schema foi acessível, mas a consulta de dados não; mapeamento é documentado e ainda não homologado.
15.2 BLOQUEANTE — Confirmar que [S2] descreve a camada atual, janela efetiva, cobertura das fontes e jurisdições, frequência, frescor e termos de uso/armazenamento. Sem isso não prometer histórico completo, completude municipal nem afirmar período exato na UI.
15.3 BLOQUEANTE — Definir representação de múltiplas categorias, limites de registros e ordenação de exibição, seleção inicial e tratamento dos status não reconhecidos; confirmar se chips e última inspeção correspondem a todos os registros exibidos. Em fonte truncada, indicar claramente contagem parcial.
15.4 BLOQUEANTE — Decidir o conflito entre a imagem (Folio, Address e Unit do permit) e a regra fixa de não duplicar exposição do Card 1; comprovar associação e escopo condomínio/unidade. Só apresentar esses campos reais após decisão compatível com as instruções do projeto; não falsificar ou remover dados da imagem por conta própria.
15.5 BLOQUEANTE — Aprovar texto de vazio de produção limitado às fontes consultadas e modo de sinalizar cobertura parcial, indisponibilidade e dados sem associação; a frase genérica da imagem isoladamente pode sugerir certeza indevida.
15.6 BLOQUEANTE PARA AMPLIAÇÃO — Encontrar e homologar meios programáticos autorizados para jurisdições municipais, code enforcement, holds, inspeções completas e certificados se esses itens forem exigidos na entrega. O menu humano [S4] por si só não prova API. Definir prioridade e custo antes de contratá-los.
15.7 Decidir se e quando comparar área/características com listing e cadastro; uma divergência é indício para revisão humana, não prova de obra ilegal. A imagem atual não contém indicador correspondente.
15.8 Confirmar com o arquiteto o uso de LLM e o mínimo permitido no contexto; não gerar conclusão legal automática. Aprovar retenção em Blob dos dados externos e privacidade com base nos termos reais.
15.9 Obter work item existente com critérios de aceite antes do PR; branch e Conventional Commit conforme regras do projeto. Reconferir bases e alterações locais; não presumir que os blueprints de Cards 6–14 foram implementados.
15.10 Referência visual já existe em legal-share/CARD12 PERMITs.png; não criar cópia. Nenhuma rota, código, API ou pesquisa real de imóvel foi executada nesta entrega.

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
