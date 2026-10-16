%md
# Diário de Decisões — Projeto CineData Analytics

Principais mudanças e decisões que tomei ao realizar a ativade e suas motivações.

## 1. Orquestração — Databricks Workflow

### 1.1. Agendamento diário às 3h da manhã (America/Sao_Paulo)

- **Contexto:** o enunciado pede um schedule simulando rotina real de produção, sem especificar horário.

Escolhi rodar, diariamente, as 3h da manhã por ser um horário de baixa concorrência para o cluster serverless, reduzindo chance de fila e de disputa por recursos. Também é antes do início do expediente, garantindo dados atualizados quando o time de BI chega.

### 1.2. Performance target: `STANDARD` (sem otimização)

- **Contexto:** o Databricks oferece `PERFORMANCE_OPTIMIZED` com custo maior.

Supus uma "escolha real de negócio" e optei por não usar a otimização do Databricks pois o pipeline é em batelada, roda 1x/dia, e os tempos reais observados (~11 min no total) são aceitáveis para o caso de uso. Otimização não é gargalo.


### 1.3. Timeouts e warnings por task (2× / 3× do tempo real, arredondado para múltiplo de 5 acima)

- **Contexto:** o Databricks permite regras de health (`RUN_DURATION_SECONDS`) e timeouts por task.

Ao observar os tempos de duração para cada task estipulei a duração de duas (2x) o tempo demorado por aquela task para o warning e de 3x para o timeout.
Bem como adicionei um tempo pra warning e timeout do Job todo, baseado na soma dos valores por task.
Obs: ao rodar outras vezes notei tempos de espera menores mas decidi manter os aqui demonstrados para segurança, não são tão grande assim.

- **Números aplicados:**

  | Task       | Tempo real | Arredondado | Warning | Timeout |
  |------------|------------|-------------|---------|---------|
  | to_Setup   | 363 s      | 365 s       | 730 s   | 1095 s  |
  | to_Bronze  | 65 s       | 65 s        | 130 s   | 195 s   |
  | to_Silver  | 102 s      | 105 s       | 210 s   | 315 s   |
  | to_Gold    | 140 s      | 140 s       | 280 s   | 420 s   |

  - Warning do Job: **1350 s**
  - Timeout do Job: **2025 s**

### 1.4. Notificação por e-mail só em falha

- **Contexto:** o Job roda diariamente; notificar sucesso todo dia vira ruído.

Só pra simular um agendamente real, adicionei a opção de enviar um e-mail notificando falha no Job.

---

## 2. Arquitetura Medalhão

### 2.1. Catálogo, schemas e volume com nomes explícitos

- **Contexto:** o enunciado permite omitir o catálogo.
Fiz como visto em aula pois achei os nomes `catalog.schema.table` mais robustos.

---

## 3. Camada Bronze

### 3.1. Leitura de CSV com todas as colunas como `string`

- **Cotexto:** `spark.read.csv(header=True)` vem com `inferSchema` por default
Optei por tirar e receber todos os dados com o tipo string só para ficar mais coerente com a camada Bronze de dados brutos.

### 3.2. `mode("overwrite")` em vez de `mode("append")`

- **Contexto:** o enunciado pede `append`.

Os 5 arquivos são uma batelada única de landing. Rodar o notebook 2x com `append` duplicaria tudo, já `overwrite` deixa a reexecução idempotente e segura.

### 3.3. Ingestão da cotação do dólar via JSON externo

- **Contexto:** Não consegui acessar `olinda.bcb.gov.br` pelo Databricks.

Para seguir a atividade a alternativa era obter o JSON externamente (mock). O tratamento do JSON (parse, tipagem, ingestão) continua sendo feito no notebook.

---

## 4. Camada Silver

### 4.1. Financeiro — interpretação de sufixos `K` e `M`

- **Contexto:** a origem tinha valores textuais com sufixos de magnitude(K e M) misturados com numéricos puros.

Utilizei um filtro com o  regex `\d\s*[KM]$` para buscar esses casos, mas garantindo que não pegaria algo como N/A.


### 4.2. Engajamento — `Column Shift` tratado por `try_cast`

Criei e usei a função `normalize_numeric` limpa separadores decimais mistos e usa `try_cast` com faixa de negócio.

- **Faixas aplicadas:**
  - Notas (TMDB e IMDb): 0–10
  - Contagem de votos: ≥ 0
  - Popularidade: 0–1500 (teto empírico)
  
Encontrei valores na casa dos milhões (inválidos) então o teto de 1500 na popularidade foi escolhido para evitar isso e pegar apenas valores válidos.