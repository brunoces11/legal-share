# LEEVLI | LEGAL ANALYSIS | CARD 9 — STRUCTURAL CONDITION AND SAFETY
# MASTER IMPLEMENTATION BLUEPRINT V1 — BASEADO NO TEMPLATE DO CARD 6 (CARD6_HOA_FINANCIAL.md, V3)
# (aderente ao código em 2026-11-10, incluindo o Plano A: coordenador de consultas por execução)

Este documento é o blueprint de implementação do Card 9 (Structural Condition and Safety).
Segue a estrutura do template do Card 6: as seções [REGRA FIXA] são idênticas às do template;
as seções [ESPECÍFICO DO CARD] são próprias do Card 9.

V1 (2026-11-10): a IMAGEM DE REFERÊNCIA do card ("Attempt B") é o contrato visual. Ela traz a
versão expandida, a compacta e o tooltip aberto. Todo dado exibido está mapeado, na seção 6.1,
para um campo real de fonte programática (cesta MLS da RealEstateAPI e ASCE Hazard Tool API,
documentação lida em 2026-11-10) ou para uma tabela pública estática versionada no código
(escala Saffir-Simpson do NHC, conferida em 2026-11-10). As seções [REGRA FIXA] foram
reconferidas contra o código em 2026-11-10: as referências de arquivo e linha datadas de
2026-11-06 continuam válidas.

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
## 6. MISSÃO, ESCOPO E FASES [ESPECÍFICO DO CARD 9]
---------------------------------------------------------------------------------------------
Card dos REGISTROS CONSTRUTIVOS e da REFERÊNCIA DE VENTO do local do imóvel: ano de
construção, idade da edificação, material da cobertura e a velocidade básica de vento de
projeto (ASCE 7) no ponto do imóvel. Todo dado vem de fonte programática e carrega a origem;
o card não avalia a estrutura, não classifica risco e não afirma conformidade.

Termos do card:
- Registro construtivo .. dado da edificação publicado pela fonte (ano, cobertura). Não é
                          laudo nem inspeção.
- Vento de referência ... velocidade básica de vento V da ASCE 7 no ponto: rajada de 3
                          segundos a 33 ft do solo, exposição C, para a Risk Category e o MRI
                          informados. É parâmetro de projeto; não é previsão, histórico de
                          furacões nem prova de resistência da edificação.
- Comparação hipotética . posição do valor de referência na escala Saffir-Simpson (NHC), que
                          é medida em vento SUSTENTADO de 1 minuto. As duas grandezas são
                          diferentes; a comparação existe só porque está no contrato visual, e
                          sempre com asterisco e nota (6.1, bloco C).

O CSV descreve exatamente esse escopo no item "Risco de furacão e vento": "Localização do
imóvel para cruzamento com mapas de risco", "Dados construtivos relevantes, como ano e
material de cobertura, quando disponíveis" e "Não retorna uma classificação individual de
risco de furacão". A OBS do item (CONTEÚDO DIVERGENTE | LINK ALTERNATIVO) aponta o ASCE Hazard
Tool como fonte de vento por endereço; NOAA e FEMA NFHL citados na linha não servem para isso.

### 6.1 CONTRATO VISUAL — IMAGEM DE REFERÊNCIA -> DADOS
Imagem recebida em 2026-11-10, variante "Attempt B", com três estados: expandido, compacto e
expandido com o tooltip da escala aberto. Os chips "Attempt B" e "Illustrative data" são
marcas do laboratório, não dados (seção 12).
Formato de cada linha: "campo (exemplo da imagem) <- fonte | estado quando ausente".

Cabeçalho
- Ícone de prédio + título "Structural Condition & Safety" (grafia da imagem, com "&") +
  seta. Sem dado.

Bloco A — "Construction records & wind reference" (painel esquerdo)
- Year built (1998)
    <- cesta MLS da RealEstateAPI, MESMA precedência do Card 1
       (legal-analysis-property-card.mapper.ts:56–61 e :94): mlsDetail.property.yearBuilt
       -> mlsSearchDetail.listing.property.yearBuilt -> mlsSearchDetail.public.yearBuilt.
       Os dois cards nunca podem mostrar anos diferentes (gate 15.2).
    |  "Unavailable"
- Building age (28 years · nota "Calculated for 2026")
    <- cálculo determinístico no backend: referenceYear - yearBuilt, com referenceYear = ano
       civil de collectedAtUtc (TimeProvider). É diferença de anos civis, não idade exata (a
       fonte não informa mês de conclusão); por isso a nota "Calculated for <ano>" é
       obrigatória e vem do payload.
    |  "Unavailable" quando não há ano válido. Ano igual ao de referência -> "Under 1 year".
       Ano maior que o de referência (obra/pré-construção) -> idade não calculada,
       "Unavailable", limitação "yearBuiltInFuture".
- Roof material (Unavailable)
    <- cesta MLS: mlsDetail.homeDetails.roof (texto do anúncio, sem tradução). Leitura igual
       à de RealEstateApiMlsDetailMapper.ReadText: string aparada; array de strings unido por
       ", "; qualquer outro tipo -> ausente.
    |  "Unavailable" (estado do exemplo da imagem). Só o MLSDetail traz homeDetails: imóvel
       resolvido só por listingId sai "Unavailable" com a limitação "roofSourceNotAvailable".

Bloco B — "Reference wind speed" (painel direito, ícone de vento)
- Valor grande "150" + unidade "MPH"
    <- ASCE Hazard Tool API, GET /v1/wind com standardsVersion=7-22 e riskLevel=2 (Risk
       Category II, residencial): entrada de wind.mriResults do MRI padrão da Risk Category
       II (700 anos), em mph. Estrutura exata das entradas -> gate 15.1.
    |  "Unavailable" (unidade oculta) quando a fonte não respondeu, não está configurada ou o
       imóvel não tem coordenadas.
- Pill "3-second gust · Exposure C · 33 ft" + botão (?)
    <- definição publicada pela própria API para wind.mriResults ("3-second gust wind speeds
       at 33 ft above ground for Exposure C Category"), gravada no payload como gustSeconds=3,
       exposure="C", heightFt=33. Some junto com o valor.
    Tooltip (?): o conteúdo não aparece na imagem; proposta para aprovar com o HTML estático
       (15.7): "ASCE <padrão> basic wind speed for Risk Category <II> (<700>-year MRI) at
       this location, from the ASCE Hazard Tool. A structural design reference, not a
       forecast." + "Retrieved <data>". Acrescenta uma frase SÓ quando o flag vier true:
       hurricaneProneRegion -> "Located in a hurricane-prone region (ASCE 7, Section 26.2).";
       specialWindRegion -> "Located in a special wind region: local wind speeds may be
       higher." Em Miami-Dade e Broward (HVHZ) acrescenta: "Local code requirements in this
       county may specify different design values." (15.4).
- Pill "If sustained 1 minute: Category 4*" + botão (?)
    <- consulta determinística do valor de referência na escala Saffir-Simpson, tabela
       estática versionada no frontend (12) com a fonte NHC. Faixas semiabertas sobre o valor
       em mph: Tropical storm [39,74) · Category 1 [74,96) · Category 2 [96,111) ·
       Category 3 [111,130) · Category 4 [130,157) · Category 5 [157,∞). Abaixo de 39 ->
       "Below tropical storm". Rótulo: "If sustained 1 minute: <faixa>*".
    |  pill oculta quando não há valor de referência.
    Tooltip (?) "Hurricane wind scale" (está na imagem):
       Tropical storm 39–73 mph · Category 1 74–95 mph · Category 2 96–110 mph ·
       Category 3 111–129 mph · Category 4 130–156 mph · Category 5 157+ mph
       Nota fixa (texto da imagem): "Hypothetical comparison for a tropical cyclone with
       1-minute sustained winds. The ASCE value is a 3-second reference gust, not a forecast
       or proof of building resistance."
       Conferido em 2026-11-10 em nhc.noaa.gov/aboutsshws.php (categorias 1–5) e
       nhc.noaa.gov/aboutgloss.shtml (tropical storm 39–73 mph, média de 1 minuto).
       A faixa correspondente ao valor fica destacada na tabela.

Rodapé — "Sources: RealEstateAPI · ASCE Hazard Tool"
    <- montado a partir de sourceStatus: só as fontes que responderam com dado. Texto fixo é
       proibido: sem resposta da ASCE, o rodapé não cita "ASCE Hazard Tool".

Versão compacta (está na imagem)
- "Structural Condition & Safety · Built <ano> · Roof <material> · Wind <valor> mph" +
  seta para baixo; cada segmento ausente mostra "Unavailable" (ex.: "Roof Unavailable").
  A comparação Saffir-Simpson NUNCA entra na linha compacta (a imagem não a mostra).

Regra de leitura do contrato: nenhum campo da imagem é fixo no card definitivo, exceto
rótulos, a tabela pública da escala e a nota do tooltip.

### 6.2 Fases
Fase 1 (este blueprint, 1 PR/1 card) = exatamente a imagem, que cobre o item do CSV "Risco de
furacão e vento":
- Cesta MLS (já em cache, sem chamada nova): ano de construção e material da cobertura.
- ASCE Hazard Tool API (todos os estados e territórios dos EUA): vento de referência e os
  flags hurricane-prone region e special wind region.
- Escala Saffir-Simpson (NHC): tabela estática para a comparação hipotética.
Fase 2 (backlog, card/PR próprio após homologação de cada fonte e aprovação de novo layout).
Itens do CSV e do PROMPT (9B/9C) que a imagem NÃO mostra — gate 15.6:
- "Milestone Inspection" (Florida DBPR; county records) — condomínio com 3+ pavimentos.
- "Recertificação 40/50 anos" (Miami-Dade ou Broward Building Department).
- "Condição insegura da estrutura" (code enforcement; fire/elevator authority).
- "SIRS e reserva estrutural": dono é o Card 6 (status de reporte DBPR); o Card 9 só o
  referencia ("SIRS: see Card 6").
- Material da cobertura pelo registro público (RealEstateAPI PropertyDetail
  propertyInfo.roofMaterial): hoje é código numérico ("180" na documentação) e a tabela de
  códigos citada pela própria RealEstateAPI (/reference/roof-materials-codes) retornou 404
  em 2026-11-10 (gate 15.5).

### 6.3 Fronteira com outros cards (regra anti-duplicidade, ⚠️ item 4)
- Card 1 (Property Identity) já EXIBE "Year built" (legal-analysis-property-card.mapper.ts:133).
  A imagem do Card 9 também o exibe, como base do cálculo de idade. CONFLITO com o ⚠️ item 4
  -> decisão 15.2. Recomendação: seguir a imagem como exceção documentada, com a mesma
  fonte e a mesma precedência do Card 1 (os valores não podem divergir). Se o arquiteto
  recusar a exceção, a linha vira referência ("Year built: see Card 1") e a idade continua
  calculada sobre o mesmo dado.
- Material da cobertura: nenhum card do relatório o exibe (Card 1, campos em
  legal-analysis-property-card.mapper.ts:131–139). O Card 9 é o DONO. A página do imóvel
  mostra "roof" em Features, fora do relatório; não conta para a regra.
- Card 5 (Flood and Environmental Risks): inundação, zona FEMA, evacuação, nível do mar.
  Nenhum dado de vento ou furacão (conferido em 2026-11-10). O Card 9 é o DONO do vento. A
  ASCE API também publica flood (FEMA NFHL): o Card 9 não a consulta.
- Card 6: SIRS. Card 12 (Permits): permits, inclusive de cobertura, e idade do telhado por
  permit. Card 14: seguro e custos. Fora do Card 9.
- Conferido em 2026-11-10: nenhum card atual (1, 3, 4, 5) exibe idade da edificação,
  material de cobertura ou vento de referência.

Proibições invariantes: sem score, nota, semáforo, "Low risk", "Hurricane-proof",
"Code compliant", "Safe" ou qualquer avaliação estrutural; a categoria Saffir-Simpson nunca é
apresentada como risco do imóvel, previsão ou histórico (só "If sustained 1 minute", com
asterisco e nota), nunca vai para a linha compacta, para o LLM ou para o Summary; o valor
ASCE nunca é convertido de rajada para sustentado; sem placeholder em produção; null nunca
vira 0; ausência de material de cobertura nunca vira "no roof issue"; o card não conclui
idade do telhado, vida útil, seguro ou necessidade de inspeção.

---------------------------------------------------------------------------------------------
## 7. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM EM 2026-11-10) [ESPECÍFICO DO CARD 9]
---------------------------------------------------------------------------------------------
- LegalAnalysisWorkerIds (Application/Contracts/LegalAnalysisContracts.cs:126–130) declara só
  PropertyDataCache e FloodEnvironmentalRisk. Os Cards 6, 7 e 8 NÃO estão implementados (só
  blueprints em legal-share). Se o Card 9 for o primeiro card novo, ele cria o switch do
  PromptBuilder (seção 5, item 5) e o helper de MLS de 3.3.
- Ano de construção na cesta: RealEstateApiMlsDetailPropertyDto.YearBuilt
  (RealEstateApiMlsDetailResponseDto.cs:76), RealEstateApiPropertyDto.YearBuilt (linha 27) e
  RealEstateApiPublicDto.YearBuilt (linha 46). O Card 1 lê na ordem mlsDetail.property ->
  mlsSearchDetail.listing.property -> mlsSearchDetail.public
  (legal-analysis-property-card.mapper.ts:56–61) e exibe "Year built" (linhas 94 e 133).
- Material da cobertura na cesta: só em mlsDetail.homeDetails.roof. HomeDetails é
  JsonElement de propósito (RealEstateApiMlsDetailResponseDto.cs:44–50) e é preservado no
  cache (IPropertyDataCacheReader devolve RealEstateApiMlsDetailDataDto, lido em
  RealEstateApiMlsDetailSource.ReadCached, linha 78). Gravações reais: "Shingle" (listing
  A12004619) e null (A12016406) em RealEstateApiOnlinePropertyDetailTests.cs:629 e :656;
  tipo divergente { "type": "Shingle" } (linha 868) vira null na leitura tolerante
  (RealEstateApiMlsDetailMapper.cs:111–118 e 154–157). RealEstateApiPropertyDto e
  RealEstateApiPublicDto não têm campo de cobertura.
- Coordenadas na cesta: mlsDetail.property.latitude/longitude
  (RealEstateApiMlsDetailResponseDto.cs:71–72), listing.property (RealEstateApiPropertyDto.cs:
  17–20) e public (RealEstateApiPublicDto.cs:22–25). O Card 5 as valida em
  FloodPropertyLocationResolver.TryCoordinate (linhas 218–229, privado) e o resolvedor devolve
  null sem coordenadas (linhas 55–57).
- Estado e condado: address.stateOrProvince e address.countyOrParish (lidos em
  FloodPropertyLocationResolver.cs:60). Normalização do nome do condado de Miami-Dade já
  existe em MiamiDadeEvacuationZoneClient (linhas 16–20); Broward não tem normalização.
- ASCE: nenhum cliente no codebase; nenhuma ocorrência de wind/hurricane no relatório.
- ConfigureEnvironmentalHttpClient (IntegrationsDependencyInjection.cs:419–426) aplica
  RemoveAllLoggers: a URL não é registrada pelos loggers do HttpClient (requisito para o token
  da ASCE, que vai na query string — seção 9.2).
- Frontend: rota card-flood (app.routes.ts:5–10) como padrão de laboratório; rawPayload é a
  união PropertyDataCacheWorkerPayload | FloodEnvironmentalRiskPayload
  (legal-analysis-result.model.ts:57); guard getFloodEnvironmentalRiskPayload
  (legal-analysis-worker-payload.ts:9–12); composição floodRiskCard em
  pages/listing-detail/components/legal-analysis-result-dialog (.ts:122; bloco @if em
  .html:64; Summary em .html:71). O Card 1 usa 'Information unavailable'
  (legal-analysis-property-card.mapper.ts:32); a imagem do Card 9 usa "Unavailable": o Card 9
  segue a imagem. Não existe componente de tooltip reutilizável nos cards.
- A seção técnica do diálogo lista "Year built" (PROPERTY_FIELDS, linha 67): é detalhamento
  técnico, não card.
- Imagem de referência do Card 9: AUSENTE em legal-share (recebida na conversa) -> gate 15.7.
- log/LegalAnalysis.txt: execução mais recente 200c0573 (2026-11-06), 44,6 s no total, com os
  dois workers atuais em menos de 2 s e o LLM em ~41 s; linha de base para medir a latência
  da ASCE (9.2).

---------------------------------------------------------------------------------------------
## 8. IDENTIDADE E APLICABILIDADE [ESPECÍFICO DO CARD 9, SEGUINDO 3.3]
---------------------------------------------------------------------------------------------
- Resolução própria do Card 9, pelos mecanismos obrigatórios de 3.3: PropertyLookup +
  IPropertyDataCacheReader; com FetchMissingSources=true, MLS ausente buscado SOMENTE por
  SharedMlsQueries (DetailAsync/ListingAsync), compartilhando a consulta com os demais workers.
- No Card 9 o registro MLS é ao mesmo tempo fonte de conteúdo (ano e cobertura) e
  pré-requisito (coordenadas, estado e condado para a ASCE).
- Campos lidos: yearBuilt (precedência de 6.1), homeDetails.roof, latitude/longitude,
  address.stateOrProvince, address.countyOrParish. Nada além disso.
- Coordenadas: mesma precedência do ano (mlsDetail.property -> listing.property -> public);
  latitude em [-90, 90], longitude em [-180, 180], finitas. Sem coordenadas válidas -> ASCE
  não consultada (unavailable, "coordinatesUnavailable"); ano e cobertura continuam.
- Não usar FloodPropertyLocationResolver: devolve null sem coordenadas e tiraria do card o
  ano e a cobertura (3.3). A validação de coordenadas do Card 5 é privada; o Card 9 faz a
  própria. Unificar é refatoração separada (registrar no PR).
- Aplicabilidade (pré-requisito, 4.1):
  * ASCE cobre os 50 estados, Porto Rico, Guam, Samoa Americana e Ilhas Virgens; fora disso ou
    resposta sem valor para o ponto -> coverageMissing;
  * HVHZ = condado Miami-Dade ou Broward (FL), normalizado como no
    MiamiDadeEvacuationZoneClient; condado ausente -> hvhz null (sem frase no tooltip);
  * o card é exibido para todo tipo de imóvel (casa, townhouse, condomínio).

---------------------------------------------------------------------------------------------
## 9. FONTES DA FASE 1 [ESPECÍFICO DO CARD 9]
---------------------------------------------------------------------------------------------
### 9.1 Cesta MLS (RealEstateAPI) — conteúdo e pré-requisito, fonte COMPARTILHADA
- Leitura pela cesta, sem alterar nenhum DTO; busca quando ausente só via SharedMlsQueries
  (3.2). Proibido chamar IOnlinePropertyDetail direto. Nenhuma chamada nova à RealEstateAPI.
- yearBuilt: inteiro (número ou string numérica) entre 1600 e referenceYear + 5; fora disso
  -> descartado com "yearBuiltInvalid". Valores diferentes entre as fontes da precedência ->
  usa o primeiro (como o Card 1) e registra "yearBuiltSourcesDiverge".
- roof: texto do provedor, até 120 caracteres, sem tradução nem normalização de categoria.
  O valor é do anúncio (CSV "Dados de anúncio e MLS": representação comercial que exige
  confirmação documental): limitação fixa "roofMaterialAsListed" sempre que exibido.
- Estado da fonte: found = ano ou cobertura presentes; noResults = registro MLS presente sem
  os dois; unavailable = sem registro MLS (falha ou não encontrado).
### 9.2 ASCE Hazard Tool API — conteúdo (vento), fonte EXCLUSIVA do Card 9 (cliente direto)
- Documentação lida em 2026-11-10: asce.org/publications-and-news/asce-hazard-tool/api,
  api-hazard.asce.org/docs (OpenAPI 3.0.3, /docs/assets/openapi.yml) e o PDF "Hazard Tool API
  Documentation" (seção Wind).
- Requisição: GET https://api-hazard.asce.org/v1/wind?lat={lat}&lon={lon}
  &standardsVersion=7-22&riskLevel=2&token={token}. Parâmetros obrigatórios: lat, lon,
  standardsVersion (7-10 | 7-16 | 7-22), riskLevel (1–4). Somente GET/HEAD.
- Resposta (PDF, seção Wind):
  * wind.mriResults: "3-second gust wind speeds at 33 ft above ground for Exposure C
    Category, based on linear interpolation between contours, in mph", para vários MRIs; MRI
    padrão por Risk Category: I 300 anos, II 700, III 1.700, IV 3.000. A saída traz todas as
    Risk Categories, mas a requisição exige uma;
  * wind.isHurricaneWindDebrisZone: hurricane-prone region (ASCE 7, Section 26.2);
  * wind.isHurricaneWindDebris1MileZone: flag interno, "funcionalmente o mesmo" que o
    anterior -> lido como OU com o anterior;
  * wind.isSpecialWindZone: special wind region (Fig. 26.5-1);
  * requestInfo.currentMonthlyRequestNumber: "usados/permitidos" (ex.: "69/1000").
  A estrutura interna de cada entrada de mriResults não está documentada para wind (o
  exemplo publicado é de snow: lista { key, value } com valor numérico em string) -> gate
  15.1. Ler por JsonElement, tolerante a número ou string; aceitar só valor finito entre 50 e
  300 mph; fora disso -> error "invalidWindValue".
- Parâmetros fixos da fase 1: standardsVersion 7-22 (edição referenciada pelo Florida
  Building Code 8ª edição, 2023 — confirmar em 15.4) e riskLevel 2 (Risk Category II,
  edificações residenciais comuns). Gravados no payload; o frontend não os supõe.
- Erros documentados: 400 entrada inválida; 401 token ausente/inválido (-> unavailable
  "notConfigured"); 404 caminho inexistente ("nunca significa 'sem dado'"; ponto sem
  cobertura devolve 200); 429 cota mensal esgotada (-> unavailable "quotaExceeded") ou taxa
  (Retry-After); 500; 502/503/504 transitórios. O corpo de erro pode ser JSON, texto ou HTML:
  decidir pelo status HTTP, nunca pelo corpo.
- Custo e cota: chave por conta, contratada por cotação (Sales Inquiry); cota mensal; só
  sucessos contam. 1 requisição por execução. Logar o currentMonthlyRequestNumber (não é
  sensível) para acompanhar a cota.
- Latência: o exemplo publicado (snow) registrou 18,6 s. SourceTimeout de 20 s na fase 1;
  medir no gate 15.1 com wind.
- Segredo: o token vai na query string por desenho da API. Fica em options (user-secrets /
  cofre), nunca no repositório, em log, em mensagem de exceção ou no payload; a exceção do
  cliente não carrega a URL; conferir que a telemetria de dependências não grava a query
  (QueryStringRedactionTelemetryProcessor). Mesmo tratamento do AuthKey do Clerk no Card 8.
- Sem cache e sem nova tentativa na fase 1. O valor de um ponto só muda com a edição da
  norma; cache em memória por (lat, lon, padrão, riskLevel) exatos só entra se o gate 15.1
  mostrar cota ou latência insuficientes E os termos de uso permitirem (decisão 15.1).
- Estado da fonte: found = valor válido; coverageMissing = 200 sem valor para o ponto;
  unavailable = não configurada, sem coordenadas, 401, 429, 5xx ou timeout; error = 400 ou
  valor inválido.
### 9.3 Escala Saffir-Simpson (NHC) — tabela estática, não é fonte
- Faixas e nota de 6.1, versionadas no frontend com a URL do NHC e a data de conferência.
  Dado interno estático: não é fonte, não entra em sourceStatus e não define o status (4.1).
### 9.4 Florida Statutes e Florida Building Code — NÃO exibidos pelo Card 9
- O CSV cita F.S. §553.73 e o FBC/HVHZ. A imagem não tem bloco de base legal: o card não
  exibe estatutos e não cria entradas no registro único (3.4) na fase 1. A diferença entre o
  valor ASCE e eventuais valores de projeto do HVHZ aparece só como frase do tooltip (6.1).

---------------------------------------------------------------------------------------------
## 10. CONTRATO DO PAYLOAD (worker "structural-condition-safety", schemaVersion 1) [ESPECÍFICO]
---------------------------------------------------------------------------------------------
{
  schemaVersion, collectedAtUtc,
  jurisdiction: { state?, hvhz?: true|false },
  construction: { status, sourceId:"realestateapi-mls",
                  yearBuilt?: { value, sourcePath:"mlsDetail"|"mlsListing"|"publicRecord" },
                  buildingAge?: { years, referenceYear, basis:"calendarYearDifference" },
                  roofMaterial?: { textRaw, sourcePath:"mlsDetail.homeDetails" } },
  windReference: { status, sourceId:"asce-hazard-tool",
                   standard:"ASCE 7-22", riskCategory:"II", mriYears?,
                   speedMph?, gustSeconds:3, exposure:"C", heightFt:33,
                   hurricaneProneRegion?, specialWindRegion?, retrievedAtUtc? },
  sourceStatus[]: formato padrão (4.1),
  limitations[]
}
- sourceId: "realestateapi-mls" | "asce-hazard-tool". status das seções: valores de
  FloodSourceStatus (4.1).
- Sem categoria Saffir-Simpson no payload: a comparação é apresentação (12) e não vai ao LLM.
- Status do worker (regra 4.1): as duas fontes são de conteúdo. Alguma found ->
  dataAvailable; senão alguma noResults/coverageMissing com resposta válida -> noResults;
  senão -> sourceUnavailable. Coordenadas, estado e condado são pré-requisito e não geram
  dataAvailable.
- Limitações (códigos fixos): roofMaterialAsListed, roofSourceNotAvailable, yearBuiltInvalid,
  yearBuiltInFuture, yearBuiltSourcesDiverge, coordinatesUnavailable, windReferenceNotForecast
  (sempre que houver valor), specialWindRegion, hvhzLocalCodeMayDiffer, quotaExceeded,
  payloadReduced.
- Sem payload bruto das fontes, sem coordenadas, sem endereço. Payload alvo <= 4 KB
  (MaxPayloadBytes 20.000); parte pesada reduzida primeiro: roofMaterial.textRaw (corte em
  120 caracteres), com "payloadReduced".
- Projeções (classe estática StructuralConditionSafetyProjection):
  * Read: valida schemaVersion, enums, faixas numéricas e coleções antes de qualquer uso;
  * ForApi: payload público do card;
  * ForReport: resumo para o LLM — yearBuilt, buildingAge, roofMaterialReported (bool, sem o
    texto), speedMph, standard, riskCategory, hurricaneProneRegion, specialWindRegion, hvhz,
    sourceStatus, limitations. Sem categoria Saffir-Simpson, sem coordenadas, sem texto livre.

---------------------------------------------------------------------------------------------
## 11. BACKEND [ESPECÍFICO DO CARD 9, SEGUINDO 3, 4 E 5]
---------------------------------------------------------------------------------------------
- Application/Contracts/StructuralConditionSafetyContracts.cs: payload tipado + porta
  IAsceWindHazardClient (GetWindAsync(lat, lon, standardsVersion, riskLevel) -> Found |
  NoCoverage + valor, MRI, flags e uso da cota; falha por exceção), com DTO próprio.
- Application/Contracts/StructuralConditionSafetyProjection.cs: Read, ForApi, ForReport.
- Application/Options/StructuralConditionSafetyOptions.cs: { SourceTimeout=20 s,
  TotalBudget=25 s, MaxPayloadBytes=20.000 }; validação no
  NewLegalAnalysisModuleDependencyInjection (SourceTimeout <= TotalBudget; TotalBudget <
  WorkerTimeout; MaxPayloadBytes < limite geral), conforme 4.1.
- Integrations.Infrastructure/Asce/Clients/AsceHazardToolClient.cs + options
  AsceHazardToolOptions { Enabled, BaseUrl, Token } com ValidateOnStart (Enabled exige BaseUrl
  https no host api-hazard.asce.org com caminho /v1 e Token preenchido). Padrão do
  RealEstateApiParcelClient: host fixo, limite de bytes da resposta, leitura por JsonDocument,
  decisão pelo status HTTP. Registro com ConfigureEnvironmentalHttpClient no
  IntegrationsDependencyInjection, sem BuildRetryPolicy e sem nova tentativa. Sem
  ProjectReference novo (já existe).
- Infrastructure/Workers/StructuralConditionSafetyWorker.cs:
  Id = LegalAnalysisWorkerIds.StructuralConditionSafety ("structural-condition-safety");
  registro explícito em AddNewLegalAnalysisWorkers. Sequência: identidade (seção 8) ->
  registros construtivos (cesta, sem rede) -> ASCE isolada com timeout -> cálculo da idade ->
  montagem -> limite de payload. Status pela regra 4.1 (seção 10). TimeProvider (também para
  referenceYear); ILogger + ILegalAnalysisRunTrace com RunId, sourceId, status, durationMs e
  currentMonthlyRequestNumber; nunca coordenadas, endereço, token, URL ou payload.
- LegalAnalysisReportPromptBuilder: switch por WorkerId (seção 5, item 5) com
  StructuralConditionSafety -> StructuralConditionSafetyProjection.ForReport; demais
  inalterados. O LLM nunca calcula idade, categoria ou conclusão estrutural.
- LegalAnalysisWorkerPayloadMapper: acrescentar StructuralConditionSafety ->
  StructuralConditionSafetyProjection.ForApi, mantendo os casos existentes.
- Não alterar: coordenador, orquestrador, cesta, clientes MLS/RentCast/Parcel/FEMA, DTOs MLS,
  RealEstateApiMlsDetailMapper, persistência, contratos HTTP, outros workers.

---------------------------------------------------------------------------------------------
## 12. FRONTEND (Angular) [ESPECÍFICO DO CARD 9, SEGUINDO 4.2 E 5]
---------------------------------------------------------------------------------------------
- data-access/models/legal-analysis-structural-payload.model.ts (payload) e
  legal-analysis-structural.model.ts (view model).
- legal-analysis-result.model.ts: StructuralConditionSafetyPayload na união de rawPayload.
- legal-analysis-worker-payload.ts: getStructuralConditionSafetyPayload (workerId
  'structural-condition-safety' + schemaVersion 1), no padrão de
  getFloodEnvironmentalRiskPayload.
- data-access/legal-analysis-structural.mapper.ts + .spec + .fixture: payload -> view model
  (textos, "Calculated for <ano>", rodapé de fontes, frases do tooltip, comparação
  Saffir-Simpson). A tabela HURRICANE_WIND_SCALE fica neste arquivo, como constante com a
  URL do NHC e a data de conferência (2026-11-10); a consulta da faixa é função pura testada
  nos limites (38, 39, 73, 74, 95, 96, 110, 111, 129, 130, 156, 157).
- ui/legal-analysis/cards/structural-condition-safety/: standalone, signals, colapsável
  (seção 5, item 11).
- Layout expandido — a imagem de referência manda (PROMPT, Regra 7):
  1) Cabeçalho: ícone de prédio + título em gradiente + seta. "Attempt B" e o chip
     "Illustrative data" só no laboratório (input sampleData, como o Summary).
  2) Linha com dois painéis (esquerdo ~47%, direito ~53%) separados por divisória vertical:
     esquerdo "Construction records & wind reference" com três linhas rótulo · valor em
     azul (Year built; Building age + nota pequena "Calculated for <ano>"; Roof material);
     direito "Reference wind speed" com ícone de vento, valor grande + "MPH" à esquerda,
     divisória vertical e duas pills empilhadas com botão (?) à direita.
  3) Rodapé "Sources: ..." em texto pequeno.
- Tooltips: (?) é <button type="button"> com aria-label, aria-expanded e aria-controls;
  abre no clique e no foco, fecha com Esc e ao sair; popover com role="dialog" e título
  ("Hurricane wind scale" no segundo). Sem hover-only (ESLint mouse-events-have-key-events e
  click-events-have-key-events). Faixa do valor destacada com texto, não só com cor.
- Linha compacta (está na imagem): "Structural Condition & Safety · Built <ano> · Roof
  <material> · Wind <valor> mph"; valores em azul; "Unavailable" quando ausente.
- Responsividade: até 900 px os dois painéis empilham e a divisória vira horizontal; até
  600 px o valor grande fica acima das pills; o popover ocupa a largura do card.
- Rota de teste: features/listings/pages/card-structural (path "card-structural", fora do
  Shell, canMatch !environment.production, loadComponent), no padrão de pages/card-flood.
  Variantes de layout só na rota de teste, todas consumindo o MESMO mock; após aprovação
  permanece apenas a variante aprovada (a imagem recebida é a "Attempt B").
- Composição no diálogo: computed structuralCard + bloco @if após o bloco do card de número
  anterior já implementado (Card 8, 7, 6 ou 5) e antes do Summary.
- Regras ESLint de template do AGENTS.md; a11y teclado/leitor de tela (ícones decorativos com
  aria-hidden; "MPH" lido como "miles per hour"). Sem PDF na fase 1.

---------------------------------------------------------------------------------------------
## 13. TESTES E ACEITE
---------------------------------------------------------------------------------------------
Backend (unit): ano pela precedência do Card 1 em todas as combinações de fontes; ano como
número e como string; ano inválido, futuro e igual ao de referência; idade com TimeProvider
fake na virada do ano; "yearBuiltSourcesDiverge"; roof string, array, objeto, null e
ausente; roof por listingId (sem mlsDetail); coordenadas válidas, ausentes e fora de faixa
(ASCE não consultada, ano e cobertura mantidos); ASCE found, sem cobertura, 400, 401, 429,
5xx, timeout, corpo HTML, valor fora de faixa; flags hurricane-prone (inclusive pelo flag de
1 milha) e special wind region; HVHZ em Miami-Dade e Broward com as grafias do condado;
orçamento/parcial com TimeProvider fake; isolamento de fonte; limite de payload; projeções
Read/ForApi/ForReport; payload mapper; prompt sem coordenadas, texto de cobertura ou
categoria Saffir-Simpson.
Backend (segurança): token ausente de log, exceção, payload e telemetria; nenhuma URL da ASCE
gravada.
Backend (reaproveitamento): MLS buscado por SharedMlsQueries (com e sem coordenador no
contexto, como em LegalAnalysisSourceCoordinatorTests); nenhuma chamada direta a
IOnlinePropertyDetail no worker; leitura pela cesta antes de buscar; nenhum uso de
FloodPropertyLocationResolver.
Backend (integração): fixture gravada do gate 15.1 (resposta real de /v1/wind para um ponto
da FL e para um ponto sem cobertura); worker no orquestrador sem afetar property-data-cache e
Flood; WebApiBoundaryTests e suíte completa.
Frontend: specs do mapper (faixas Saffir-Simpson nos limites; "Under 1 year"; rodapé
dinâmico; frases condicionais do tooltip), do componente (expandido/compacto/tooltips por
teclado/parcial/sem dados) e dos guards; mesmo "Year built" do Card 1 para o mesmo payload de
cesta; nenhum texto "safe", "compliant" ou "risk" renderizado; smoke desktop/mobile com
capturas no work item, comparadas com a imagem de referência.
Homologação real: condomínio em Miami-Dade (HVHZ) com roof no MLS; casa fora do HVHZ na FL;
imóvel resolvido só por listingId (roof "Unavailable"); imóvel sem coordenadas; ASCE sem
token; ASCE com cota esgotada. Conferir no log/LegalAnalysis.txt os eventos Worker e
SourceQuery, a duração da ASCE e o uso da cota.
Aceite: todos os campos da imagem dinâmicos e com fonte; o mesmo ano do Card 1; cards já
implementados e Summary inalterados; nenhum dado de outro card repetido além da exceção
aprovada em 15.2; nenhum valor simulado; categoria nunca fora do tooltip/pill hipotética;
execução total < 120 s.

---------------------------------------------------------------------------------------------
## 14. ORDEM DE IMPLEMENTAÇÃO
---------------------------------------------------------------------------------------------
1) Validação de dados e fontes: este documento + conferência anti-duplicidade + decisões
   15.2, 15.3 e 15.6 + gate 15.1 (contratação da ASCE: iniciar já).
2) Rota card-structural com HTML estático nas versões expandida, compacta e com tooltips,
   fiel à imagem; aprovação visual (inclui o texto do tooltip da primeira pill, 15.7).
3) Worker com a cesta MLS (ano, idade, cobertura): options, registro no DI, projeções,
   PromptBuilder, PayloadMapper, modelo/guard/mapper/componente Angular, composição no
   diálogo. O painel de vento sai "Unavailable" com sourceStatus "notConfigured".
4) ASCE Hazard Tool como incremento aditivo, quando o 15.1 estiver confirmado: só preenche
   windReference no mesmo payload; o card não muda de forma.
5) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile, execução real e leitura do log.

---------------------------------------------------------------------------------------------
## 15. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
---------------------------------------------------------------------------------------------
15.1 ASCE Hazard Tool API: pedir cotação (Sales Inquiry) e obter a chave; confirmar cota
     mensal, custo e limite de taxa; ler os termos de uso (exibição no relatório com
     atribuição "ASCE Hazard Tool", armazenamento do valor no pacote de auditoria do Blob,
     cache em memória). Gravar UMA resposta real de /v1/wind (7-22, riskLevel 2) para um
     ponto da Flórida e uma para um ponto sem cobertura e anotar: estrutura exata de
     mriResults (chaves e qual entrada é o MRI de 700 anos), tipos dos valores, presença dos
     flags, resposta sem cobertura e latência. Bloqueia o passo 4.
15.2 Duplicidade "Year built" com o Card 1 (⚠️ item 4). Recomendação: aprovar como exceção
     documentada, porque a imagem é o contrato e o ano é a base visível do cálculo de idade;
     condição: mesma fonte e mesma precedência do Card 1, coberta por teste. Alternativa:
     "Year built: see Card 1".
15.3 Comparação Saffir-Simpson (produto/jurídico): a imagem compara um valor de rajada de 3 s
     com uma escala de vento sustentado de 1 minuto. Recomendação: aprovar como está na imagem
     (rótulo "If sustained 1 minute", asterisco e nota fixa), só no card expandido, fora do
     payload, do LLM e do Summary, porque o CSV proíbe "classificação individual de risco de
     furacão".
15.4 Parâmetros de vento: confirmar ASCE 7-22 como edição do Florida Building Code vigente
     (8ª edição, 2023) e Risk Category II para residencial; confirmar no FBC (capítulo 16,
     disposições HVHZ) se Miami-Dade e Broward fixam valores de projeto próprios e o texto da
     frase do tooltip.
15.5 Cobertura pelo registro público: a tabela de códigos de roof material da RealEstateAPI
     (/reference/roof-materials-codes, citada na Property Search) deu 404 em 2026-11-10.
     Quando publicada, e se o Card 8 tiver integrado /v2/PropertyDetail, avaliar
     propertyInfo.roofMaterial como fonte alternativa; a fonte passa a ser compartilhada e
     entra no coordenador com a chave LegalAnalysisSourceKeys.PropertyDetail (3.2).
15.6 Escopo: confirmar que a fase 1 é só o que a imagem mostra (item "Risco de furacão e
     vento") e que Milestone Inspection, recertificação 40/50 e unsafe structure vão para a
     fase 2 com nova imagem. O PROMPT (9F) dá relevância 9/10 a esses itens em condomínio.
15.7 Imagem de referência: salvar em legal-share (padrão CARD9_STRUCTURAL_IMG_REF.<ext>);
     aprovar o texto do tooltip da primeira pill (6.1); informar se existe variante além da
     "Attempt B".
15.8 Uso de LLM no card: em aberto (arquiteto Bruno, PROMPT 9E). O worker entrega fatos; o LLM
     só recebe ForReport.
15.9 Base de código (reconferido em 2026-11-10): d_leevli-platform-api está em
     feat/legal-analysis-source-coordinator (57074dd6 "Modulariza Workers"), que contém o
     Card 5 e o Plano A; d_leevli-platform-web está em LEGAL (71e79913). Os Cards 6, 7 e 8
     não estão implementados: as peças compartilhadas (switch do PromptBuilder, helper de MLS
     de 3.3) são criadas por quem for implementado primeiro; os outros só acrescentam. A
     branch do card parte da branch de integração que contenha Card 5 + Plano A.
15.10 Work item com critérios de aceite; branch feat/<card>-structural-condition-safety;
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
