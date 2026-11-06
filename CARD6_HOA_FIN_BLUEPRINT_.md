# LEEVLI | LEGAL ANALYSIS | CARD 6 — HOA FINANCIAL EXPOSURE
# MASTER IMPLEMENTATION BLUEPRINT V1 (aderente ao código em 2026-11-02)

## 0. MISSÃO, ESCOPO E FASES
Card único e consolidado da associação (HOA/COA): todo dado financeiro útil aparece uma
única vez, vindo de fonte verificável — API estruturada OU documento/dataset público estável
com URL. A origem (API x documento) é só detalhe arquitetural; para o usuário vale: útil,
relevante, confiável, implementável. Variantes de layout mostram exatamente o mesmo view model.

Fase 1 (este blueprint, 1 PR/1 card), conforme PROMPT_DEV_LEGAL.txt §6 e o CSV
Fontes_API_Legal_analyses.csv:
  - "Existência de HOA ou associação" -> MLS (RealEstateAPI) + Florida DBPR (condomínios)
  - "Taxa mensal da associação" -> MLS: valor, frequência, 2ª associação, itens incluídos
    ("as listed"; CSV: valor no MLS pode estar desatualizado)
  - "SIRS e reserva estrutural" -> Florida DBPR SIRS reporting (somente condomínio; só o
    status de reporte, nunca valores de reserva)
  - Base legal financeira -> Florida Statutes oficiais (Online Sunshine), por regime
Fase 2 (backlog, card/PR próprio após homologação de cada fonte): "Special assessment",
"Saúde financeira da associação" (budget, reservas, inadimplência, dívidas), "Seguro master
do prédio", estoppel (valores e cobranças da unidade), histórico de taxas, "Litígios da
associação" (County Clerk Civil Dockets, por condado; o CSV só cita
Miami-Dade, a jurisdição deve ser resolvida antes), RentCast property records (hoa.fee) como
fonte cruzada. Motivo: dependem de documentos privados da associação (sem URL pública
permanente) ou de fonte não integrada; não há fluxo de recebimento de documentos no
NewLegalAnalysis (mesma lacuna do Card 15).
Fora do Card 6: regras de uso/locação/pets (Card 7), Milestone Inspection (Card 9),
liens de condomínio/HOA §718.116/§720.3085 registrados (Card 10), CDD (Card 11),
FHA Condo (Card 14).

Proibições invariantes: sem HOA Health Score, "Low Risk", gauge, pesos, % de funding,
saldo de reservas ou qualquer número não retornado por fonte; sem placeholder em produção;
null nunca vira 0; taxa ausente != "sem HOA"; ausência de SIRS na lista DBPR != estudo
inexistente ou dispensado; valor MLS sempre rotulado "as listed" com data do anúncio;
responsável por pagamento nunca presumido.

## 1. PONTOS DE INTEGRAÇÃO EXISTENTES (NÃO ALTERAR SEMÂNTICA)
- Orquestração: RunLegalAnalysisHandler -> ILegalAnalysisOrchestrator (workers paralelos,
  contexto imutável, sem dependência entre workers, sem Buildings.Id).
- Limites: WorkerTimeout 60 s; RunTimeout 120 s compartilhado com o LLM; MaxPayloadBytes
  1.000.000 por worker; MaxInputBytes do prompt 750.000; MaxConcurrentRuns 1.
- Persistência: Building -> Blob (workers.json + report.json + manifest.json) -> PropertyReport.
  NÃO criar arquivo novo no pacote nem migration/tabela.
- API: LegalAnalysisWorkerPayloadMapper publica hoje só property-data-cache.
- Prompt: LegalAnalysisReportPromptBuilder hoje envia rawPayload de TODOS os workers (sem
  projeção por workerId; o Card 5 propõe criá-la).
- Frontend: LegalAnalysisResultDialogComponent; ordem atual property-information,
  recent-sales, active-listings, summary (mock). Cards 1–5 e Summary não mudam.
- Seção técnica do diálogo já lê 'associationFee' (PROPERTY_FIELDS) por busca de chave.

## 2. ESTADO REAL DO CÓDIGO (CONFERIDO ITEM A ITEM)
- Cache: PropertyDataCacheReader serializa DTOs TIPADOS (camelCase), não o JSON bruto.
  Campos não declarados no DTO são descartados antes do cache.
- mlsSearchDetail (RealEstateApiMlsListingItemDto -> listing.property = RealEstateApiPropertyDto):
  contém AssociationFee (decimal?) — único VALOR de taxa disponível hoje. Sem nome,
  2ª associação ou itens incluídos.
- mlsDetail (RealEstateApiMlsDetailDataDto, /v2/MLSDetail): a classe tipada não declara
  campo HOA, mas HomeDetails (JsonElement) é preservado no cache e serializado para o
  payload do property-data-cache. As gravações reais usadas em
  RealEstateApiOnlinePropertyDetailTests.cs (listings A12004619 e A12016406) mostram
  data.homedetails.associationFeeFrequency (nome CONFIRMADO; valor null nas duas
  gravações, nunca visto preenchido). Valor e frequência vêm de respostas diferentes
  (mlsSearchDetail x mlsDetail): o worker precisa lê-los das duas fontes e rotular a origem.
- RealEstateAPI MLS Search v3 documenta filtros has_hoa, hoa_amenities,
  hoa_association_name, hoa_association_name2, hoa_fee_min/max, hoa_fee2_min/max,
  hoa_fee_frequency, hoa_fee_frequency2 (docs lidas em 2026-11-02). São filtros de
  request: indicam que o dataset tem esses atributos, mas os NOMES no response não estão
  confirmados -> gate 9.1.
- RentCast: IRentCastClient só tem AVM, rent e market; /v1/properties (hoa.fee, "HOA Fee
  Amount (Monthly)", documentado) NÃO integrado.
- DBPR: nenhum client. Páginas do CSV retornaram 403 ao fetch automatizado -> gate 9.2.
- Divergências encontradas FORA do escopo (não corrigir neste PR, reportar ao arquiteto):
  PropertyDetailDto.Hoa (decimal?) sem mapeamento de origem; property-detail.mapper.ts
  tipa hoa como boolean; building-overview-section usa defaults estáticos ('$385/mo',
  lista de includes) na página do imóvel.

## 3. IDENTIDADE
- Mesma resolução do Card 5: registro MLS em cache (IPropertyDataCacheReader) e, com
  FetchMissingSources=true, IOnlinePropertyDetail (cache-aside). Reutilizar o PropertyLookup
  interno compartilhado proposto no Card 5 (Infrastructure/Workers); o PR que entrar
  primeiro faz a extração, sem alterar comportamento (testes atuais intactos).
- Campos: endereço, unidade, city, state, ZIP, county, propertyType/subType, subdivisão
  (SubdivisionName), nome(s) da associação quando o gate 9.1 confirmar.
- Regime da associação: "condominium" só com match DBPR inequívoco; caso contrário
  "unconfirmed". Townhouse/propertyType NÃO define regime (CSV). Fora da Flórida:
  DBPR e estatutos FL -> notApplicable; MLS continua.

## 4. FONTES DA FASE 1
### 4.1 RealEstateAPI MLS (primária para cobranças)
- Hoje, sem alterar nenhum DTO: valor = mlsSearchDetail.listing.property.associationFee;
  frequência = mlsDetail.homeDetails.associationFeeFrequency (leitura tolerante de
  JsonElement, mesmo estilo de FindString do worker; texto/null).
- Nome(s) da associação, 2ª associação e itens incluídos: só entram após o gate 9.1
  (declarar nos DTOs MLS apenas campos confirmados no response real, nullable e aditivo;
  afeta Properties/Integrations: rodar testes desses módulos).
- Regras: frequência ausente (inclusive null, o caso das duas gravações) -> exibir valor com
  "frequency not reported" e NÃO anualizar; anualizado = valor x fator da frequência
  publicada, usando somente os valores confirmados no gate 9.1 (esperados: Monthly 12,
  Quarterly 4, Semi-Annually 2, Annually 1); qualquer outro valor -> não calcular, exibir original.
  Segunda associação exibida separada, nunca somada sem frequência confirmada das duas.
  "Fee includes" exibe valores originais do provedor, sem tradução por suposição.
  Data de referência = ModificationTimestamp/ListingContractDate do anúncio.
### 4.2 Florida DBPR — registro de condomínios
- Gate 9.2 antes do código: confirmar URL do dataset público, formato, colunas (nome da
  associação, nº do projeto/arquivo, condado, unidades, administradora, endereço, status)
  e periodicidade, a partir do egress de produção.
- Match: nome da associação (MLS) + condado + endereço/ZIP normalizados. 0 candidatos ->
  noResults ("No matching DBPR condominium record"; não significa "não é condomínio");
  >1 sem desempate -> ambiguous, exibir só a contagem; nunca escolher arbitrariamente.
- Se for arquivo bulk: espelho local versionado (snapshot com retrievedAtUtc, URL de origem
  e hash), carregado em memória com limite; atualização manual na fase 1, rotina de
  verificação de atualização na fase 2. Nunca baixar o arquivo por execução.
### 4.3 Florida DBPR — SIRS reporting (condomínio confirmado em 4.2)
- Mesmo gate 9.2: lista pública de reportes SIRS (formato/campos/atualização).
- Exibir: "SIRS completion reported to DBPR" + data quando houver registro; sem registro ->
  "No SIRS report found in the DBPR list" + limitação (não prova ausência/dispensa).
  Nunca exibir valores de reserva/funding (CSV: só quando divulgados no estudo -> fase 2).
### 4.4 Florida Statutes (referências legais financeiras)
- Registro interno estático e versionado no código (sem banco): seção, heading oficial,
  URL Online Sunshine, ano do estatuto ("2026 Florida Statutes"), regime aplicável.
  Confirmadas em 2026-11-02: §718.116 "Assessments; liability; lien and priority...";
  §720.30851 "Estoppel certificates"; §718.112 "Bylaws" (subseções (2)(f)-(g) budget,
  reservas e SIRS); §720.303 "Association powers and duties; ... official records; budgets;
  financial reporting...". Incluir também §718.111(11)-(13) (seguro, registros, relatório
  financeiro) após conferir heading/URL.
- Regime condominium -> cap. 718; regime unconfirmed -> 718 e 720 com rótulo "applies
  depending on the association's legal regime". Texto exibido: só heading oficial + link;
  sem interpretação jurídica nem conclusão sobre o imóvel.
- Esta seção é a ponte para a fase 2: cada estatuto lista o documento que o rege
  (estoppel, budget, financial report, SIRS/reserve study, insurance), como "Documents
  governed by this section", sem afirmar que existem ou que o comprador tem direito a eles.

## 5. CONTRATO DO PAYLOAD (worker "hoa-financial-exposure", schemaVersion 1)
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
  legalReferences[]: { section, heading, url, statuteYear, regime, governs[] },
  notCoveredByPublicSources[]: códigos fixos (specialAssessments, reserves, budget,
                               delinquency, masterInsurance, litigation, estoppel),
  sourceStatus[]: { sourceId, applicable, queried, status, durationMs, recordsCount?,
                    errorClass? },
  limitations[]
}
- status enum igual ao Card 5: found | noResults | notApplicable | unavailable |
  coverageMissing | ambiguous | error. Atualidade separada do status.
- Sem payload bruto do provedor; payload alvo <= 30 KB.

## 6. BACKEND (arquitetura existente)
- NewLegalAnalysis.Application/Contracts: IFloridaDbprCondominiumClient (registro + SIRS,
  DTOs próprios). MLS reutiliza IPropertyDataCacheReader/IOnlinePropertyDetail.
- Integrations.Infrastructure/FloridaDbpr: implementação; ProjectReference conforme o
  precedente definido no Card 5; rodar WebApiBoundaryTests.
- IntegrationsDependencyInjection: AddOptions<T>().Bind().Validate().ValidateOnStart();
  AddHttpClient sem BuildRetryPolicy; no máximo 1 retry 5xx/timeout.
- NewLegalAnalysis.Infrastructure/Workers/HoaFinancialExposureWorker:
  Id = LegalAnalysisWorkerIds.HoaFinancialExposure ("hoa-financial-exposure"); registro
  explícito em AddNewLegalAnalysisWorkers. HoaFinancialExposureOptions
  { SourceTimeout=8 s, TotalBudget=20 s } validado < WorkerTimeout. Registro estático de
  estatutos como classe interna do worker/módulo (sem DI extra).
  Status do worker: dataAvailable se qualquer seção found; noResults se todas noResults;
  sourceUnavailable se todas falharem. TimeProvider; ILogger + ILegalAnalysisRunTrace sem
  endereço/nome de associação/payload.
- PromptBuilder: se a projeção do Card 5 existir, enviar só resumo (regime, charges,
  status DBPR/SIRS, sourceStatus); senão o payload pequeno entra como está. LLM nunca
  calcula anualizado nem decide regime.
- PayloadMapper: publicar hoa-financial-exposure (payload público por desenho).

## 7. FRONTEND (Angular)
- models/legal-analysis-result.model.ts: HoaFinancialExposurePayload na união
  discriminada de rawPayload (compartilhada com o Card 5; ajustar narrowing existente).
- data-access/legal-analysis-hoa-financial.mapper.ts: payload -> view model (labels,
  moeda, frequência, omissões).
- ui/legal-analysis/cards/hoa-financial-exposure/: standalone, signals, padrão Regra 4
  (collapsed = signal(false), botão aria-expanded, paths 'm5 12 5-5 5 5'/'m5 7 5 5 5-5').
- Layout expandido (um bloco por dado, sem repetição):
  1) Cabeçalho: título + nome(s) da associação + regime (ou "Association regime unconfirmed").
  2) Painel verde "Financial Snapshot" (somente fatos): taxa principal "as listed",
     frequência, anualizado (se calculável), 2ª associação, status SIRS. Sem nota/score.
  3) Fee details: includes, data do anúncio.
  4) Association registry (DBPR): projeto, condado, unidades, administradora.
  5) Florida law references: heading + link oficial + "Documents governed".
  6) Rodapé: "Not covered by public sources" (lista fixa) + fontes/datas/limitações.
- Linha compacta: "HOA Financial Exposure · $<fee> <frequência> (as listed) ·
  <associação ou 'Association unconfirmed'> · SIRS: <reported date | not found | n/a>".
- Rota de teste: features/listings/pages/card-hoa-fin (path "card-hoa-fin", fora do Shell,
  canMatch !environment.production, loadComponent), diálogo com mock no padrão
  card-builder. Variantes de layout podem coexistir só na rota de teste, todas consumindo
  o MESMO mock; após aprovação permanece apenas a variante aprovada.
- Composição: após o Card 5 (ou após Active Listings, se o Card 5 ainda não existir) e
  antes do Summary.
- Regras ESLint de template do AGENTS.md; a11y teclado/leitor de tela. Sem PDF na fase 1.

## 8. TESTES E ACEITE
Backend (unit): fee com/sem frequência, frequências não mapeadas, 2ª associação, null !=
0, regime por match DBPR (0/1/N candidatos), SIRS found/noResults/notApplicable, fora da
FL, sem MLS, orçamento/parcial com TimeProvider fake, registro de estatutos por regime,
payload mapper, prompt sem dados sensíveis.
Backend (integração): fixtures gravadas (MLS condo com HOA, DBPR, SIRS); worker no
orquestrador sem afetar property-data-cache; DTOs MLS estendidos sem quebrar Properties;
WebApiBoundaryTests e suíte completa.
Frontend: specs do mapper, componente (expandido/compacto/parcial/sem dados) e mappers
ajustados; smoke desktop/mobile com capturas no work item.
Homologação real: condomínio em Miami-Dade com HOA no MLS e registro DBPR; casa em HOA
(regime unconfirmed); imóvel sem taxa no MLS; DBPR indisponível; caso ambíguo.
Aceite: Cards 1–5 inalterados; nenhum valor simulado; todo dado com fonte/data; nenhuma
ausência tratada como "sem associação/sem cobrança"; execução total < 120 s.

## 10. ORDEM DE IMPLEMENTAÇÃO (decisão técnica do blueprint)
1) Validação de dados e fontes (Regra 1): este documento + confirmação do 9.1/9.2.
2) Rota card-hoa-fin com HTML estático nas versões expandida e compacta (Regra 3/4),
   usando só campos que a fase 1 entrega; aprovação visual.
3) Worker com MLS (valor + frequência) + registro de estatutos: entrega útil sem nenhuma
   integração nova; registrar no DI, mapper público, modelo/mapper/componente Angular,
   composição no diálogo.
4) DBPR (registro + SIRS) como incremento aditivo, quando o 9.2 estiver confirmado: só
   acrescenta seções ao mesmo payload; o card não muda de forma.
5) Suíte completa, WebApiBoundaryTests, smoke desktop/mobile.

## 9. PRÉ-REQUISITOS, GATES E DECISÕES DO ARQUITETO
9.1 Conferência de campos (feita pela equipe antes de codar, não por este blueprint):
    obter UMA resposta real de MLSDetail/MLSSearch de um condomínio com HOA e anotar os
    nomes exatos dos campos de nome da associação, 2ª associação, itens incluídos e os
    valores reais de frequência. Sem isso, a fase 1 sai só com valor + frequência
    (associationFee + homeDetails.associationFeeFrequency); o restante fica omitido.
    Não bloqueia o início: Seções 4.1 e 10 já definem o que é construído antes.
9.2 Confirmar, do egress de produção, dataset DBPR de condomínios e lista SIRS (fetch
    automatizado recebeu 403). Bloqueia 4.2/4.3; sem eles o card sai só com MLS + estatutos.
9.3 Decidir: espelho local do DBPR (local de armazenamento e rotina manual) vs consulta
    online, conforme o formato confirmado.
9.4 Decidir: integrar RentCast /v1/properties (hoa.fee mensal) na fase 1 como fonte
    cruzada (custo por chamada) ou manter na fase 2.
9.5 Uso de LLM no card: em aberto (arquiteto Bruno).
9.7 Observação de fonte: as gravações de referência citam Central HQ
    (20-ARQUITETURA/APIs/responses/realestateapi/02-mlsdetail.json), fora deste repositório;
    não foi lida nesta análise.
9.6 Work item com critérios de aceite; branch feat/<card>-hoa-financial-exposure a partir de
    develop; commits feat(new-legal-analysis): #<card> ...
