##  Relatório Técnico de Engenharia de Desempenho (Performance Testing)

**Data:** Novembro de 2025

**Engenheiro de Performance:** Wanessa Dias

**Projeto:** `ecommerce-checkout-api` (Simulado)

---

### 1. Resumo Executivo: Capacidade Máxima da API

A suíte de testes de performance executada utilizando **k6** revelou os seguintes limites de capacidade para os diferentes perfis de carga da `ecommerce-checkout-api`:

| Cenário de Carga | Endpoint | Perfil da Operação | Capacidade Máxima Estimada | Ponto de Ruptura (Stress) |
| :--- | :--- | :--- | :--- | :--- |
| **I/O Bound** | `/checkout/simple` | Leve (I/O, DB) | **~300 VUs** | O Spike Test falhou ao atingir 300 VUs. |
| **CPU Bound** | `/checkout/crypto` | Pesada (CPU, Hash) | **< 999 VUs** | O Stress Test demonstrou falhas massivas antes de 1000 VUs. |

O **Teste de Carga** para o endpoint `/checkout/simple` com 50 VUs foi bem-sucedido e manteve as latências dentro do SLA estabelecido. No entanto, o **Teste de Estresse** e o **Teste de Pico** demonstraram que a aplicação tem dificuldades significativas em lidar com saltos ou aumentos progressivos de carga acima de um determinado ponto, culminando em uma falha massiva no teste de estresse.

---

### 2. Evidências dos Testes de Execução (k6)

Abaixo estão os resumos de execução do k6 para cada um dos quatro testes realizados, comprovando os resultados e métricas coletadas.

#### 2.1. Teste de Smoke (`tests/smoke.js`)

**Objetivo:** Verificar a saúde básica e a estabilidade da API com carga mínima.

| Métrica | Valor Observado | Critério de Sucesso | Status |
| :--- | :--- | :--- | :--- |
| `checks_succeeded` | 100.00% | 100% | ✅ **Sucesso** |
| `http_req_duration` | avg: 1ms | - | - |

**Evidência:**


#### 2.2. Teste de Carga (`tests/load.js`)

**Objetivo:** Simular o pico de 50 usuários simultâneos no endpoint I/O (`/checkout/simple`) e verificar o atendimento do SLA.

| Métrica | Valor Observado | SLA (Threshold) | Status |
| :--- | :--- | :--- | :--- |
| `http_req_failed` | 0.00% | $\leq 1\%$ | ✅ **Aprovado** |
| `http_req_duration p(95)` | 295.37ms | $\leq 500ms$ | ✅ **Aprovado** |

**Evidência:**


#### 2.3. Teste de Pico (`tests/spike.js`)

**Objetivo:** Simular um aumento de carga abrupto (Flash Sale) de 10 para 300 VUs no endpoint I/O (`/checkout/simple`).

| Métrica | Valor Observado | Análise | Status |
| :--- | :--- | :--- | :--- |
| `checks_failed` | 0.00% | Falha de Conexão Crítica (Protocol Error) | ❌ **Falha** |
| `http_req_failed` | 0.00% | - | - |
| **⚠️ Erro Crítico** | `dial tcp 127.0.0.1:3000: connect: connection refused` | O servidor não conseguiu responder a novas conexões no pico. | ❌ **Falha** |

**Evidência:**


#### 2.4. Teste de Estresse (`tests/stress.js`)

**Objetivo:** Aumentar a carga agressivamente no endpoint CPU Bound (`/checkout/crypto`) para encontrar o ponto de ruptura.

| Métrica | Valor Observado | Análise | Status |
| :--- | :--- | :--- | :--- |
| `checks_failed` | 99.56% (1,167,262 falhas) | **Falha Massiva** | ❌ **Falha** |
| `http_req_duration max` | $\approx 1m0s$ | Timeouts Extremos (Latência Ilimitada) | - |
| `vUs max` | 1000 | Carga Máxima Atingida | - |

**Evidência:**


---

### 3. Análise de Estresse: Identificação do Ponto de Ruptura

O Teste de Estresse, que utilizou o endpoint **CPU-heavy** (`/checkout/crypto`), revelou o ponto de ruptura da aplicação. O cenário foi desenhado para aumentar progressivamente a carga de 0 a 1000 VUs em 6 minutos.

* **Carga Agressiva:** O cenário aumentou a carga de 0 a 200, 200 a 500, e 500 a 1000 VUs.
* **Comportamento:** Inicialmente, o sistema manteve uma taxa de sucesso aceitável.
* **Ponto de Ruptura (Breaking Point):** A falha massiva ocorreu na transição do segundo para o terceiro estágio (após 500 VUs), e tornou-se catastrófica ao se aproximar de **1000 VUs**.

**Indicação da Falha:**

1.  A taxa de falha (**`checks_failed`**) atingiu **99.56%**.
2.  O **tempo de resposta máximo** (`http_req_duration max`) se tornou $\approx 1m0s$ (indicando **Timeouts** extremos e requisições presas).
3.  A taxa de requisições por segundo (**Throughput**) caiu drasticamente, demonstrando que o servidor estava sobrecarregado e não conseguia mais processar as requisições em tempo hábil.

Isso sugere que a arquitetura ou o provisionamento de recursos da API, especialmente para operações que consomem muita CPU, não é escalável para atender a centenas de usuários simultâneos, falhando massivamente **antes de atingir a carga total de 1000 VUs**.