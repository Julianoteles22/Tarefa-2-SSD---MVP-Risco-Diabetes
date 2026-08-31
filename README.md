# MVP1 de SSD - Classificação do Risco de Diabetes

**Disciplina:** Sistemas de Suporte à Decisão
**Universidade de Brasília (UnB)**
**Aluno:** Juliano Teles Abrahao - 231013411

Este projeto apresenta um MVP de classificação do **risco de diabetes**, utilizando uma base de 15.000 pacientes com informações **demográficas, de estilo de vida, antropométricas, de sinais vitais e de exames laboratoriais**.

## Objetivo

Treinar e comparar modelos de Machine Learning para classificar o risco de diabetes de cada paciente em 3 categorias:

- **Baixo**
- **Moderado**
- **Alto**

Mais do que atingir a maior acurácia possível, o MVP busca responder a uma pergunta de decisão: **é possível estratificar o risco de um paciente antes de pedir exames laboratoriais?**

## Definição do Problema

O projeto é tratado como um problema de **classificação multiclasse**.

A variável `diabetes_risk` já vem rotulada na base (`Low`, `Moderate`, `High`) e é convertida para um código ordinal:

| Código | Classe | Proporção na base |
|---|---|---|
| 0 | Baixo | 60,0% |
| 1 | Moderado | 25,0% |
| 2 | Alto | 15,0% |

A base é **desbalanceada**, o que exige atenção à métrica de avaliação e ao balanceamento das classes durante o treino.

## Hipótese

Características de **estilo de vida, perfil socioeconômico, medidas antropométricas e sinais vitais** contêm informação suficiente para separar pacientes de risco baixo, moderado e alto **antes** da realização de exames laboratoriais.

Para testar essa hipótese, o notebook compara dois cenários de modelagem sobre exatamente a mesma divisão treino/teste:

| Cenário | Features | Aplicação prática |
|---|---|---|
| **Triagem** | Sem exames laboratoriais (questionário, medidas e sinais vitais) | Priorizar quem deve ser encaminhado para exame |
| **Clínico** | Com `fasting_blood_sugar` e `hba1c_level` | Estratificar risco após a coleta laboratorial |

A diferença entre os dois cenários **quantifica o ganho de informação trazido pelos exames**.

## Variáveis Utilizadas

### Features categóricas (ambos os cenários)

- `gender`, `city`, `family_history_diabetes`, `physical_activity_level`, `diet_type`, `smoking_status`, `alcohol_consumption`, `income_bracket`

### Features numéricas — cenário Triagem

- `age`, `bmi`, `hours_sleep_per_night`, `stress_level`, `waist_circumference_cm`, `blood_pressure_systolic`, `blood_pressure_diastolic`

### Features numéricas — cenário Clínico

- todas as anteriores **+** `fasting_blood_sugar` e `hba1c_level`

### Target

- `risco_codigo` (0 = Baixo, 1 = Moderado, 2 = Alto)

O modelo **não recebe** `patient_id` (identificador sem valor clínico) nem `diabetes_risk` / `classe_nome` (o próprio alvo em outro formato).

## Tratamento dos Dados

### Limpeza

- remoção de pacientes duplicados por `patient_id`;
- regras de plausibilidade clínica: `age` entre 18 e 95, `bmi > 0`, `fasting_blood_sugar > 0`, `hba1c_level > 0` e pressão sistólica maior que a diastólica;
- descarte de linhas com alvo ou medidas numéricas nulas.

Resultado: **15.000 → 14.998 linhas** (2 registros com pressão sistólica ≤ diastólica).

### Valores ausentes

| Variável | % ausente | Padrão |
|---|---|---|
| `alcohol_consumption` | 25,25% | **MNAR** |
| `smoking_status` | 3,15% | MCAR |
| `income_bracket` | 3,09% | MCAR |

A ausência em `alcohol_consumption` **não é aleatória**: 41,3% entre mulheres contra 10,3% entre homens, e 43,5% acima dos 60 anos contra ~24,5% nas demais faixas. Descartar essas linhas removeria um subgrupo específico e enviesaria a base.

Por isso, a ausência é **tratada como categoria própria** (`Nao informado`), preservando todos os registros e permitindo que o modelo aprenda se o próprio "não informar" carrega sinal.

## Modelos Comparados

1. **Dummy Classifier** — baseline mínimo (prevê sempre a classe majoritária)
2. **Regressão Logística Multinomial** — baseline supervisionado e interpretável
3. **Random Forest** — modelo principal
4. **Random Forest otimizado** com `GridSearchCV` (5 folds, otimizando F1 Macro)

Os modelos supervisionados usam `class_weight="balanced"` para compensar o desbalanceamento.

## Métricas de Avaliação

- Acurácia
- Precisão ponderada
- Recall ponderado
- F1-Score ponderado
- F1-Score Macro
- Matriz de confusão
- Análise de overfitting

A métrica principal é o **F1 Macro**, pois atribui o mesmo peso às três classes. Prever sempre "Baixo" já entrega 60% de acurácia sem acertar um único paciente de risco Alto — exatamente o grupo que mais importa em um sistema de apoio à decisão clínica.

## Resultados

Conjunto de teste: 3.000 pacientes (20%, divisão estratificada).

| Cenário | Modelo | Acurácia | F1 Ponderado | **F1 Macro** |
|---|---|---|---|---|
| Triagem | Dummy Baseline | 0,600 | 0,450 | 0,250 |
| Triagem | Regressão Logística | 0,526 | 0,540 | 0,439 |
| Triagem | Random Forest | 0,592 | 0,580 | 0,462 |
| Clínico | Dummy Baseline | 0,600 | 0,450 | 0,250 |
| Clínico | Regressão Logística | 0,767 | 0,776 | 0,737 |
| Clínico | Random Forest | 0,774 | 0,781 | 0,738 |
| Clínico | **Random Forest otimizado** | **0,775** | **0,782** | **0,738** |

Melhores hiperparâmetros: `n_estimators=400`, `max_depth=None`, `min_samples_leaf=1` (F1 Macro de 0,744 na validação cruzada).

### Desempenho por classe — Random Forest otimizado

| Classe | Precisão | Recall | F1 | Suporte |
|---|---|---|---|---|
| Baixo | 0,90 | 0,83 | 0,86 | 1.800 |
| Moderado | 0,55 | 0,68 | 0,60 | 750 |
| Alto | 0,80 | 0,70 | 0,75 | 450 |

Apenas **3 pacientes de risco Alto foram classificados como Baixo** — o erro mais custoso em triagem clínica. Praticamente todos os erros ocorrem nas fronteiras Baixo↔Moderado e Moderado↔Alto.

### Variáveis mais importantes

| Variável | Importância acumulada |
|---|---|
| `fasting_blood_sugar` | 0,259 |
| `hba1c_level` | 0,217 |
| `city` | 0,066 |
| `age` | 0,065 |
| `bmi` | 0,058 |
| `waist_circumference_cm` | 0,056 |

Correlação com o risco: `fasting_blood_sugar` 0,74 e `hba1c_level` 0,73, contra 0,29 de `bmi` e `age`.

> A importância de `city` é um artefato: a variável gera 18 colunas no one-hot e a importância por impureza favorece variáveis com muitos valores distintos. A análise exploratória não mostra diferença relevante de risco entre cidades.

## Principais Conclusões

1. **Os exames laboratoriais são determinantes.** O F1 Macro salta de 0,46 (Triagem) para 0,74 (Clínico) — um ganho de 60%. As duas variáveis laboratoriais concentram quase metade da importância do modelo.

2. **Sem exames, a triagem tem poder preditivo limitado.** Os modelos de Triagem superam claramente o baseline em F1 Macro (0,46 contra 0,25), mas ficam longe do cenário Clínico. O questionário e as medidas de consultório **não substituem** a coleta laboratorial; servem como priorização de fila.

3. **Acurácia engana em base desbalanceada.** No cenário de Triagem, a Regressão Logística tem acurácia (0,526) **abaixo** do Dummy (0,600) e ainda assim um F1 Macro muito superior (0,439 contra 0,250). Com `class_weight="balanced"` o modelo troca acertos na classe majoritária por acertos em Moderado e Alto — o comportamento desejado em triagem.

4. **A otimização trouxe pouco ganho.** O F1 Macro praticamente não mudou após o `GridSearchCV`, indicando que o limite do problema com estas features já havia sido atingido. O `GridSearchCV` escolheu a floresta sem restrição de profundidade (acurácia de treino de 0,9999 contra 0,775 em teste), mas ela ainda foi a melhor na validação cruzada — comportamento típico de Random Forest, em que a média sobre 400 árvores controla a variância.

## Estrutura do Repositório

```text
mvp_classificacao_risco_diabetes/
├── MVP1_SSD_Classificacao_Risco_Diabetes.ipynb
├── diabetes_risk.csv
├── README.md
├── requirements.txt
└── .gitignore
```

A base de dados (`diabetes_risk.csv`, 15.000 registros, ~1,5 MB) está **versionada junto com o repositório**, o que torna o projeto reprodutível sem download externo.

## Como Executar

O notebook detecta automaticamente o ambiente e localiza a base: se `diabetes_risk.csv` estiver no diretório de trabalho, ele é carregado direto; caso contrário, no Colab é solicitado o upload.

### Google Colab (recomendado)

Clonar o repositório traz a base junto. Na primeira célula de código do Colab:

```bash
!git clone https://github.com/<usuario>/<repositorio>.git
```

Depois, `%cd <repositorio>`, abra `MVP1_SSD_Classificacao_Risco_Diabetes.ipynb` e execute as células em ordem.

Alternativa: abrir o notebook direto no Colab e fazer upload de `diabetes_risk.csv` quando solicitado.

### Localmente

```bash
pip install -r requirements.txt
```

Execute o notebook a partir da pasta do repositório — a base será encontrada automaticamente.

A célula de `GridSearchCV` executa 90 ajustes (18 combinações × 5 folds) e é a mais demorada do notebook — alguns minutos, dependendo da máquina.

## Requisitos da Base

A base deve conter as colunas:

`patient_id`, `age`, `gender`, `city`, `bmi`, `family_history_diabetes`, `physical_activity_level`, `diet_type`, `smoking_status`, `alcohol_consumption`, `hours_sleep_per_night`, `stress_level`, `fasting_blood_sugar`, `hba1c_level`, `blood_pressure_systolic`, `blood_pressure_diastolic`, `waist_circumference_cm`, `income_bracket`, `diabetes_risk`.

## Aplicações

- **priorização de exames** em unidades com capacidade laboratorial limitada;
- **estratificação de risco** de carteiras de pacientes em saúde populacional;
- **alertas clínicos** para pacientes cujo risco previsto é Alto;
- **planejamento de campanhas de prevenção** direcionadas aos perfis de maior risco.

## Limitações

- A base é **sintética**, gerada a partir de um modelo estatístico conhecido. As relações encontradas refletem esse processo gerador e **não devem ser interpretadas como evidência clínica**.
- Não há informação temporal: é um retrato único de cada paciente, sem acompanhamento.
- O rótulo `diabetes_risk` é uma **estratificação de risco**, não um diagnóstico confirmado de diabetes.
- Faltam variáveis relevantes na literatura, como medicação em uso, comorbidades, histórico de glicemia e diabetes gestacional.
- A ausência não aleatória em `alcohol_consumption` foi tratada como categoria, o que preserva a base, mas mantém o viés de subnotificação embutido no dado.

## Próximos Passos

- testar Gradient Boosting / XGBoost / LightGBM;
- avaliar SMOTE e outras técnicas de reamostragem para as classes minoritárias;
- ajustar o limiar de decisão para maximizar o recall da classe Alto;
- aplicar SHAP para explicabilidade paciente a paciente;
- validar o modelo em uma base real e externa antes de qualquer uso clínico;
- disponibilizar o modelo em uma interface de consulta para novos pacientes.

## Fonte dos Dados

Base pública obtida no Kaggle e incluída neste repositório como `diabetes_risk.csv`.

Os dados são **sintéticos**, gerados por um script de simulação que modela correlações entre idade, IMC, atividade física, estresse, exames laboratoriais e risco de diabetes, incluindo ruído irredutível (para evitar separação perfeita) e padrões realistas de valores ausentes — MCAR em `smoking_status` e `income_bracket`, MNAR em `alcohol_consumption`.

---

**Aluno:** Juliano Teles Abrahao
**Matrícula:** 231013411
**Disciplina:** Sistemas de Suporte à Decisão
**Universidade de Brasília (UnB)**
