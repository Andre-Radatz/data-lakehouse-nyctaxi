# Data Lakehouse — NYC Taxi Trips

Pipeline de dados construído no Databricks seguindo a arquitetura Medallion 
(Bronze/Silver/Gold), com orquestração automatizada e validação de qualidade 
de dados integrada.

## Arquitetura
samples.nyctaxi.trips (Unity Catalog)

BRONZE --> Bronze_nyctaxi <br><br>
Ingestão bruta, sem tratamento<br>
Preserva o dado original para auditoria/reprocessamento<br>
filtros de validade, remoção de duplicatas<br>

SILVER --> Silver_nyctaxi <br><br>
Dado limpo, particionado por mês (pickup_month)<br>
Schema padronizado, pronto para consumo analítico<br>
agregação diária, métricas de negócio<br>

Teste --> Teste_de_Qualidade<br><br>
8 validações automatizadas<br>
Falha o pipeline se alguma regra de negócio for violada<br>

## Stack técnica

| Componente        | Escolha                     | Por quê                                                      |
|--------------------|------------------------------|----------------------------------------------------------------|
| Compute            | Databricks (PySpark)         | Processamento distribuído gerenciado, sem overhead de infra   |
| Armazenamento      | Delta Lake (tabelas gerenciadas Unity Catalog) | ACID transactions, time travel, schema enforcement |
| Orquestração       | Databricks Workflows         | Nativo da plataforma, sem infra adicional para manter          |
| Particionamento    | `pickup_month` na Silver     | Otimiza consultas filtradas por período sem gerar small files |
| Qualidade de dados | Validações PySpark customizadas | Controle total da lógica; evolução natural para Great Expectations |

## Estrutura do repositório

Data-lakehouse-nyctaxi/<br>
├── 00.Inicio                          # Confirmação de catálogo/schema no Unity Catalog<br>
├── 01.Bronze_Extração                 # Ingestão bruta de samples.nyctaxi.trips<br>
├── 02.Silver_Transformação            # Limpeza, padronização e particionamento<br>
├── 03.Gold_Pronto                     # Agregação de faturamento diário<br>
├── 04.Teste_De_Qualidade              # Validações automatizadas de qualidade<br>
└── README.md

## Como rodar

1. Criar um workspace Databricks (Free Edition ou superior)
2. Confirmar catálogo/schema executando `00.Inicio`
3. Executar os notebooks em ordem: `01` → `02` → `03` → `04`
4. (Opcional) Importar como Databricks Workflow para execução automatizada 
   e agendada — ver seção de orquestração abaixo

## Orquestração

O pipeline está configurado como um Databricks Job (`pipeline_lakehouse_nyctaxi`) 
com dependências explícitas entre as 4 tasks, agendado para rodar diariamente, 
com notificação por e-mail em caso de falha.

## Decisões técnicas e trade-offs

- **Tabelas gerenciadas em vez de tabelas externas com caminho explícito**: 
  optei por `saveAsTable()` em vez de apontar `LOCATION` para um caminho físico. 
  Isso delega o gerenciamento de armazenamento ao Unity Catalog, evitando 
  problemas de resolução de credenciais em ambientes com governança mais restrita.
- **Particionamento aplicado apenas na Silver**: a Bronze é lida por completo 
  na maioria dos reprocessamentos (não se beneficia de poda de partição), e a 
  Gold já é pequena e agregada (particionar geraria partições de uma linha só, 
  um anti-padrão conhecido como small file problem).
- **Validações de qualidade customizadas em vez de um framework pronto**: 
  para este estágio do projeto, PySpark puro dá controle total sobre a lógica 
  e evita uma dependência externa.

## Possíveis evoluções

- Migrar validações de qualidade para Great Expectations ou dbt tests
- Adicionar CDC (Change Data Capture) para ingestão incremental em vez de overwrite completo
- Explorar Liquid Clustering como alternativa ao particionamento tradicional
- Adicionar testes automatizados no CI/CD via GitHub Actions