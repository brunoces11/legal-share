# LEEVLI | LEGAL ANALYSIS | CARD 5 — FLOOD & ENVIRONMENTAL RISK
# MASTER IMPLEMENTATION BLUEPRINT V7 (aderente ao código em 2026-11-02)

## 0. MISSÃO, ESCOPO E FASES
Fase 1 (este blueprint, 1 PR/1 card): escopo aprovado no guia (PROMPT_DEV_LEGAL.txt §5)
e no CSV Fontes_API_Legal_analyses.csv:
  - "Flood zone oficial" -> FEMA NFHL
  - "Proximidade ambiental" / "Contaminação ambiental" -> EPA ECHO + Florida DEP
  - "Sinkhole e riscos geológicos" -> FGS Subsidence (somente Casa/Townhouse, conforme CSV)
  - "Elevação e risco de inundação" -> exibir Elevation Certificate somente quando houver fonte
    homologada para a jurisdição; na fase 1 não há fonte mapeada para Miami-Dade/Broward -> omitir.
Fase 2 (backlog, cada item com card/PR próprio após homologação): NOAA climate normals mensais
(estender NceiClimateNormalsClient), USGS sismos, USGS/NWPS, NOAA Storm Events, HURDAT2,
NIFC, Pinellas EC (só se o mercado incluir Pinellas), geometria de parcela, reabertura/PDF
do relatório (lacuna geral, não exclusiva do Card 5). Vento/HVHZ (CSV "Risco de furacão e
vento", fonte sugerida ASCE Hazard Tool) não pertence ao Card 5; avaliar no Card 9.

Proibições invariantes: sem score, sem gauge, sem pesos, sem "safe/risk-free", sem
placeholder em produção, ausência de registro != ausência de risco, proximidade != impacto.

## 1. PONTOS DE INTEGRAÇÃO EXISTENTES (NÃO ALTERAR SEMÂNTICA)
- Orquestração: RunLegalAnalysisHandler -> ILegalAnalysisOrchestrator (workers paralelos,
  contexto imutável, sem dependência entre workers, sem Buildings.Id).
- Limites: WorkerTimeout 60 s; RunTimeout 120 s compartilhado com o LLM; MaxPayloadBytes
  1.000.000 por worker; MaxInputBytes do prompt 750.000; MaxConcurrentRuns 1.
- Persistência: Building -> Blob (workers.json + report.json + manifest.json) -> PropertyReport.
  Validador do pacote exige exatamente esses 3 arquivos: NÃO criar arquivo novo.
- API: LegalAnalysisWorkerPayloadMapper publica somente projeções revisadas.
- Frontend Angular 22: LegalAnalysisResultDialogComponent compõe os cards;
  ordem atual: property-information, recent-sales, active-listings, summary (mock).
- Mapas: Google Maps já usado em features/neighborhood; não adicionar ArcGIS SDK.
- Cards 1–4 e o Summary não mudam de comportamento.

## 2. IDENTIDADE E LOCALIZAÇÃO
- Fonte: registro MLS em cache (IPropertyDataCacheReader) e, em run com
  FetchMissingSources=true, IOnlinePropertyDetail (cache-aside), mesmo fluxo de
  PropertyDataCacheWorker. Extrair PropertyLookup para tipo interno compartilhado em
  Infrastructure/Workers, sem alterar comportamento (testes atuais devem passar intactos).
- Campos: latitude/longitude (RealEstateApiMlsDetailResponseDto: Latitude/Longitude),
  endereço, city, state, ZIP, county, propertyType. Confirmar o caminho JSON real no
  registro cacheado antes de codificar.
- Escopo espacial: "property point (listing coordinates)". Sem parcela na fase 1:
  nunca afirmar classificação do lote inteiro.
- Sem lat/lon válidos: não consultar fontes espaciais; seção location status
  "locationUnavailable"; card exibe "Location unavailable for this listing".
- Fora da Flórida: DEP e FGS notApplicable; FEMA e EPA continuam.

## 3. FONTES DA FASE 1
### 3.1 FEMA NFHL (primária)
- https://hazards.fema.gov/arcgis/rest/services/public/NFHL/MapServer
- Gate de homologação ANTES do código: chamar ?f=json a partir do egress de produção;
  confirmar IDs/nome das camadas (Flood Hazard Zones, FIRM Panels, LOMR, LOMA) e campos.
  IDs 28/3/1/34 são candidatos, não constantes.
- Consulta: point intersect, inSR/outSR 4326, outFields explícitos (FLD_ZONE, ZONE_SUBTY,
  SFHA_TF, STATIC_BFE, DEPTH, V_DATUM, LEN_UNIT; painel: FIRM_PAN, EFF_DATE).
- Tratar "error" dentro de HTTP 200 como falha.
- LOMA/LOMR: exibir só identificador/data quando o ponto intersecta o polígono; nunca
  concluir alteração de zona.
- Regras: BFE sempre com unidade e datum; DEPTH só para AO (ausente -> "Data unavailable";
  outras zonas -> "Not applicable"); Zone D = perigo não determinado; resposta vazia !=
  Zone X -> status coverageMissing.
### 3.2 FEMA fallback (Esri Living Atlas "USA Flood Hazard Reduced Set")
- Usar somente se a primária falhar/bloquear; rótulo visível "SFHA-only dataset".
- Contém FLD_ZONE, ZONE_SUBTY, SFHA_TF, STATIC_BFE, V_DATUM, DEPTH, LEN_UNIT; NÃO contém
  painel FIRM nem data efetiva -> esses campos "Unavailable from fallback source".
- Vazio no fallback = "not in a mapped SFHA polygon in this dataset", nunca "Zone X"/"No".
- Não reutilizar IFloodHazardClient do Neighborhood (contrato local e Sfha=false no vazio).
### 3.3 EPA ECHO
- echo_rest_services.get_facilities (p_lat, p_long, p_radius=1) + paginação por QueryID
  conforme documentação vigente; validar parâmetros/campos no ato.
- Exibir até N=10 mais próximos + total ("116 facilities within 1 mi" é cenário real).
- Texto: "EPA-regulated facility nearby"; programas/status só se publicados.
### 3.4 Florida DEP Cleanup (CLEANUP_SP/MapServer/0)
- distance=1609 m, units esriSRUnit_Meter, outFields explícitos; campos confirmados ao vivo:
  DEP_CLEANUP_SITE_KEY, SOURCE_DATABASE_NAME/ID, CPAC_PROGRAM_AREA_ID,
  CLCC_CLEANUP_CATEGORY_KEY, RSC2_REMEDIATION_STATUS_KEY, DATA_LOAD_DATE, BUSINESS_NAME,
  ADDRESS1, CITY, ZIP5, CALC_COORD_ACCURACY_LEVEL_ID, LATITUDE_DD, LONGITUDE_DD.
- Códigos sem dicionário confirmado: exibir valor original, não traduzir por suposição.
- Deduplicação com ECHO: apenas por ID oficial em comum; sem ID, manter separados.
### 3.5 FGS Subsidence (FGS_SUBSIDENCE/MapServer/0)
- Aplicável só a Casa/Townhouse (CSV). Tipo desconhecido -> notApplicable
  (reason=propertyTypeUnconfirmed), sem consulta.
- distance=8047 m; campos confirmados: REF_NUM, EVENT_DATE, DATE_REV, TRUE_SINK, ACCURACY,
  RPT_SOURCE, LATDD, LONGDD, COUNTY.
- Texto: "Reported subsidence nearby"; TRUE_SINK preservado; nunca "sinkhole confirmed"
  sem TRUE_SINK confirmado pelo dicionário da camada.

## 4. CONTRATO DO PAYLOAD (worker "flood-environmental-risk", schemaVersion 1)
{
  schemaVersion, collectedAtUtc,
  location: { latitude, longitude, source:"listing", scope:"propertyPoint",
              county?, state?, status },
  sections: {
    floodMapping:  { status, freshness, source, zone?, zoneSubtype?, sfha?, bfe?{value,unit,
                     datum}, depth?{value,unit}|notApplicable, firmPanel?, effectiveDate?,
                     revisions[], limitations[] },
    environmentalFacilities: { status, freshness, completeness{shown,total,truncated},
                               records[] },
    environmentalCleanup:    { mesmo formato },
    geologicalEvents:        { mesmo formato }
  },
  map: { center, geometry? (GeoJSON recortado/generalizado), legendZones[],
         attribution, geometryOmittedReason? },
  sourceStatus[]: { sourceId, applicable, queried, status, durationMs, recordsCount?,
                    errorClass? },
  limitations[]
}
- status enum (por seção/fonte): found | noResults | notApplicable | unavailable |
  coverageMissing | ambiguous | error. Atualidade (retrievedAtUtc, effectiveDate) e
  completude (total/truncated) separadas do status.
- record: { sourceId, providerRecordId, name, category?, status?, eventDate?,
            distanceMeters (haversine), coordinateAccuracyRaw?, datasetUrl, retrievedAtUtc }.
- Null nunca vira 0. Sem payload bruto do provedor.
- Geometria: envelope ~±0,01° + maxAllowableOffset; teto 150 KB; acima -> omitir
  geometria com geometryOmittedReason. Payload total alvo <= 300 KB.

## 5. BACKEND (arquitetura existente)
- NewLegalAnalysis.Application/Contracts: interfaces IFemaFloodMapClient,
  IEpaEchoClient, IFloridaDepCleanupClient, IFloridaSubsidenceClient (DTOs próprios).
- Integrations.Infrastructure/{Fema,Epa,FloridaDep}: implementações; helper interno único
  para query ArcGIS (montagem de URL, encoding, erro-em-200) usado por FEMA/DEP/FGS.
  Adicionar ProjectReference Integrations.Infrastructure -> NewLegalAnalysis.Application
  (precedente: Neighborhood/Experts.Application). Rodar WebApiBoundaryTests.
- IntegrationsDependencyInjection: AddOptions<T>().Bind().Validate().ValidateOnStart() por
  fonte; AddHttpClient SEM BuildRetryPolicy (3 retries/~8 s incompatível); no máximo 1
  retry para 5xx/timeout dentro do orçamento.
- NewLegalAnalysis.Infrastructure/Workers/FloodEnvironmentalRiskWorker:
  Id = LegalAnalysisWorkerIds.FloodEnvironmentalRisk ("flood-environmental-risk");
  registro explícito em AddNewLegalAnalysisWorkers.
  Orçamento: FloodEnvironmentalRiskOptions { SourceTimeout=8 s, TotalBudget=20 s }
  validado < WorkerTimeout; fontes em paralelo; ao atingir o orçamento, retornar parcial
  (status unavailable/timedOut por fonte). Status do worker: dataAvailable se qualquer
  seção found; noResults se todas noResults; sourceUnavailable se todas falharem.
  TimeProvider para horários; ILogger + ILegalAnalysisRunTrace sem payload/endereço.
- LegalAnalysisReportPromptBuilder: projeção explícita por workerId; para
  flood-environmental-risk enviar só resumo (zone, sfha, bfe, contagens, sourceStatus).
  Nunca geometria/records ao LLM.
- LegalAnalysisWorkerPayloadMapper: publicar flood-environmental-risk (payload já é público
  por desenho); demais workers continuam null.
- Persistência: nenhuma migration/tabela. Dados auditáveis no workers.json do pacote atual.
  Refresh = nova execução (botão existente). Sem novas rotas na fase 1.

## 6. FRONTEND (Angular)
- models/legal-analysis-result.model.ts: FloodEnvironmentalRiskPayload; rawPayload passa a
  união discriminada por workerId; ajustar narrowing em market-card.mapper.ts e
  legal-analysis-property-card.mapper.ts (comportamento idêntico).
- data-access/legal-analysis-flood-risk.mapper.ts: payload -> view model (labels humanos,
  unidades, regras de omissão/N/A).
- ui/legal-analysis/cards/flood-environmental-risk/: componente standalone com signals;
  expandido (3 colunas de dados + mapa) e compactado (linha de campos presentes); estado de
  expand/collapse local, sem nova requisição.
- Composição no diálogo: após o bloco recent-sales/active-listings e antes do summary
  (preserva "Card 5 após Active Listings" do guia sem mover cards existentes).
- Mapa: reutilizar o carregamento Google Maps do módulo neighborhood; GeoJSON via Data
  layer, pin do ponto, legenda só de zonas presentes; sem geometria -> só pin +
  "Flood overlay unavailable".
- Regras ESLint de template do AGENTS.md; a11y por teclado/leitor de tela.
- Sem PDF na fase 1 (export atual cobre só o Property Card).

## 7. EXIBIÇÃO
found -> exibir com fonte/data; noResults opcional -> omitir ou "No matching records within
<raio> in <dataset>"; notApplicable -> omitir seção (exceto Flood Depth "Not applicable");
coverageMissing/unavailable/error -> sem valor, aviso discreto no rodapé de fontes;
fallback FEMA -> rótulo "SFHA-only dataset". Sem blocos vazios. Fontes e limitações
sempre acessíveis.

## 8. TESTES E ACEITE
Backend (unit): mapeamento FEMA (AE com BFE, AO com/sem DEPTH, D, vazio primário,
vazio fallback), erro-em-200, haversine/raios, truncamento ECHO, aplicabilidade FGS por
tipo, fora da FL, sem lat/lon, orçamento/parcial com TimeProvider fake, teto de geometria,
projeção do prompt sem geometria, payload mapper.
Backend (integração): fixtures gravadas das consultas reais; worker no orquestrador sem
afetar property-data-cache; WebApiBoundaryTests e suíte completa.
Frontend: specs do mapper, do componente (expandido/compactado/parcial/sem dados) e dos
mappers ajustados; smoke desktop/mobile com capturas no work item.
Homologação real: imóvel em SFHA AE (ex.: Miami Beach), imóvel Zone X, casa vs condomínio
(FGS), fonte indisponível, FEMA primária bloqueada -> fallback.
Aceite: Cards 1–4 inalterados; nenhum valor simulado; toda informação com fonte/data;
nenhuma ausência tratada como segurança; execução total < 120 s com 1 retry do LLM.

## 9. PRÉ-REQUISITOS E BLOQUEIOS
- Testar FEMA NFHL a partir do egress de produção (bloqueia 3.1; 3.2 cobre parcialmente).
- Corrigir Blob 403 local para afirmar auditoria ponta a ponta (não bloqueia exibição).
- Work item com critérios de aceite; branch feat/<card>-flood-environmental-risk a partir
  de develop; commits feat(new-legal-analysis): #<card> ...
