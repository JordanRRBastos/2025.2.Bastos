
# 📚 Documentação Técnica: Power BI & Machine Learning

Este documento contém todos os scripts e fórmulas utilizados no desenvolvimento deste projeto. Abaixo estão detalhadas as Medidas DAX, Colunas Calculadas, transformações no Power Query (Linguagem M) e o script Python utilizado para a integração do modelo de Machine Learning.

-----

## 📊 1. Medidas DAX (Measures)

Indicadores de performance e agregações dinâmicas.

### Concluídos \< 3 dias

```dax
Concluídos < 3 dias = 
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    FILTER(
        'FAT_CERTIFICADOS',
        'FAT_CERTIFICADOS'[Fase atual] = "Concluído"
            && 'Medidas'[Média Lead Time Total (Emitidos)] <= 3
    )
)
```

### % Concluídos \< 3 dias

```dax
% Concluídos < 3 dias = 
DIVIDE(
    [Concluídos < 3 dias],
    [Total Solicitações Concluídas],
    0
)
```

### Média Lead Time Agendamento (Arquivados)

```dax
Média Lead Time Agendamento (Arquivados) = 
 AVERAGEX(
    FAT_CERTIFICADOS,
    DATEDIFF(FAT_CERTIFICADOS[Primeira vez que entrou na fase Agendamento], FAT_CERTIFICADOS[Primeira vez que entrou na fase Arquivado], DAY)
 )
```

### Média Lead Time Agendamento (Emitidos)

```dax
Média Lead Time Agendamento (Emitidos) = 
 AVERAGEX(
    FAT_CERTIFICADOS,
    DATEDIFF(FAT_CERTIFICADOS[Ultima vez que entrou na fase Agendamento], FAT_CERTIFICADOS[Primeira vez que entrou na fase Concluído], DAY)
 )
```

### Média Lead Time Total (Arquivados)

```dax
Média Lead Time Total (Arquivados) = 
 AVERAGEX(
    FAT_CERTIFICADOS,
    DATEDIFF(FAT_CERTIFICADOS[Criado em], FAT_CERTIFICADOS[Primeira vez que entrou na fase Arquivado], DAY)
 )
```

### Média Lead Time Total (Emitidos)

```dax
Média Lead Time Total (Emitidos) = 
 AVERAGEX(
    FAT_CERTIFICADOS,
    DATEDIFF(FAT_CERTIFICADOS[Criado em], FAT_CERTIFICADOS[Primeira vez que entrou na fase Concluído], DAY)
 )
```

### Tickets Ativos Contagem (Somente 1s)

```dax
Tickets Ativos Contagem (Somente 1s) = 
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    'FAT_CERTIFICADOS'[solicitacao ativa] = 1
)
```

### Tickets Atrasados Vencidos (Contagem)

```dax
Tickets Atrasados Vencidos (Contagem) = 
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    'FAT_CERTIFICADOS'[solicitacao ativa] = 1,
    'FAT_CERTIFICADOS'[Status_Risco_SLA_Coluna] = "Atrasado Vencido (Real)"
)
```

### Tickets em Risco Alto

```dax
Tickets em Risco Alto = 
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    'FAT_CERTIFICADOS'[solicitacao ativa] = 1, // Filtra apenas tickets ativos
    'FAT_CERTIFICADOS'[SLA_Previsto] = 1 // Previsto para atrasar (ou já atrasado)
)
```

### Tickets em Risco Alto (Contagem)

```dax
Tickets em Risco Alto (Contagem) = 
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    'FAT_CERTIFICADOS'[solicitacao ativa] = 1,
    'FAT_CERTIFICADOS'[Status_Risco_SLA_Coluna] = "Alto Risco (Previsto)"
)
```

### Total Solicitações

```dax
Total Solicitações = CALCULATE(
    COUNT('FAT_CERTIFICADOS'[Código]),
            USERELATIONSHIP(DIM_CALENDARIO[Data],'FAT_CERTIFICADOS'[Criado em]))
```

### Total Solicitações Arquivadas

```dax
Total Solicitações Arquivadas = CALCULATE(
    COUNT('FAT_CERTIFICADOS'[Código]),
    'FAT_CERTIFICADOS'[Fase atual] = "Arquivado",
            USERELATIONSHIP(DIM_CALENDARIO[Data],'FAT_CERTIFICADOS'[Primeira vez que entrou na fase Arquivado]))
```

### Total Solicitações Concluídas

```dax
Total Solicitações Concluídas = CALCULATE(
    COUNT('FAT_CERTIFICADOS'[Código]),
    'FAT_CERTIFICADOS'[Fase atual] = "Concluído",
            USERELATIONSHIP(DIM_CALENDARIO[Data],'FAT_CERTIFICADOS'[Primeira vez que entrou na fase Concluído]))
```

### Total Solicitações em Agendamento

```dax
Total Solicitações em Agendamento = CALCULATE(
    COUNT('FAT_CERTIFICADOS'[Código]),
    'FAT_CERTIFICADOS'[Fase atual] = "Agendamento"
       )
```

-----

## 🧮 2. Colunas Calculadas (DAX)

Colunas criadas diretamente no modelo de dados para fins de segmentação e regras de negócio.

### SLA\_Dias\_Uteis

```dax
SLA_Dias_Uteis = 
VAR DataInicial = FAT_CERTIFICADOS[Criado em]
VAR DataFinal   = FAT_CERTIFICADOS[Última vez que entrou na fase Concluído]
RETURN
COUNTROWS(
    FILTER(
        'DIM_CALENDARIO',
        'DIM_CALENDARIO'[Data] >= DataInicial &&
        'DIM_CALENDARIO'[Data] <= DataFinal &&
        'DIM_CALENDARIO'[FinalSemana] = "Não" &&
        'DIM_CALENDARIO'[FERIADO] = "Não"
    )
)-1
```

### SLA\_Atrasado\_3Dias\_Corrigido\_Coluna

```dax
SLA_Atrasado_3Dias_Corrigido_Coluna = 
VAR RawValue = 'FAT_CERTIFICADOS'[SLA_Atrasado_3Dias]
RETURN
    IF(
        RawValue = 10,
        1,
        RawValue
    )
```

### Probabilidade\_Atraso\_Final\_Coluna

```dax
Probabilidade_Atraso_Final_Coluna = 
VAR RawProb = 'FAT_CERTIFICADOS'[Probabilidade_Atraso] 
VAR MaxValidProbScale = 100 
VAR DefaultOutlierValue = 0.45 

RETURN
    IF(
        RawProb > MaxValidProbScale,
        DefaultOutlierValue, 
        DIVIDE(RawProb, 100) 
    )
```

### Idade Cliente Corrigida\_Coluna

```dax
Idade Cliente Corrigida_Coluna = 
DIVIDE('FAT_CERTIFICADOS'[Idade Cliente], 10)
```

### Status\_Risco\_SLA\_Coluna

```dax
Status_Risco_SLA_Coluna = 
VAR ProbFinal = 'FAT_CERTIFICADOS'[Probabilidade_Atraso_Final_Coluna]
VAR ColunaSolicitacaoAtiva = 'FAT_CERTIFICADOS'[solicitacao ativa] 
VAR DataEntrada = 'FAT_CERTIFICADOS'[Criado em] 
VAR LimiteSLADias = 3 
VAR DataLimiteSLA = IF(NOT ISBLANK(DataEntrada), DataEntrada + LimiteSLADias, BLANK()) 
VAR DataAtual = TODAY()

RETURN
    IF(
        ColunaSolicitacaoAtiva = 1 && NOT ISBLANK(DataEntrada) && DataLimiteSLA < DataAtual, 
        "Atrasado Vencido (Real)", 
        
        IF(
            ProbFinal >= 0.7, 
            "Alto Risco (Previsto)",
            IF(
                ProbFinal >= 0.4, 
                "Médio Risco (Previsto)",
                "Baixo Risco (Previsto)"
            )
        )
    )
```

-----

## 🔄 3. Colunas Adicionadas (Power Query / M)

Transformações e lógicas aplicadas durante a etapa de ETL.

### SLA\_Atrasado\_3Dias

```powerquery
let
    DataDeEntrada = [Criado em], // Sua coluna de data de entrada (ajuste o nome se for diferente)
    DataDeConclusao = [Última vez que entrou na fase Concluído], // Sua coluna de data de conclusão (ajuste o nome se for diferente)
    LimiteSLADias = 3, // Seu SLA fixo em dias

    SLA_Status_Original =
        if DataDeEntrada = null or DataDeConclusao = null then null // Não é possível saber se atrasou se não foi concluído
        else
            let
                DiasGastos = Duration.Days(DataDeConclusao - DataDeEntrada)
            in
                if DiasGastos > LimiteSLADias then 1 else 0
in
    SLA_Status_Original
```

### Idade Cliente

```powerquery
let
    DataNascimentoRaw = [Data de Nascimento], // Coluna de origem
    DataAtual = Date.From(DateTime.LocalNow()),
    
    // Converte a DataNascimento para tipo Date, lidando com nulos e erros
    DataNascimento = try Date.From(DataNascimentoRaw) otherwise null,

    CalculaIdade = 
        if DataNascimento = null or DataNascimento > DataAtual then
            null // Retorna null se DataNascimento for nula/inválida ou futura
        else
            let
                AnoNascimento = Date.Year(DataNascimento),
                MesNascimento = Date.Month(DataNascimento),
                DiaNascimento = Date.Day(DataNascimento),
                
                AnoAtual = Date.Year(DataAtual),

                // Determina a data do aniversário neste ano, ajustando para 29 de fevereiro
                AniversarioEsteAno =
                    if MesNascimento = 2 and DiaNascimento = 29 and Date.DaysInMonth(#date(AnoAtual, 2, 1)) = 28 then
                        // Se for 29/Fev e o AnoAtual NÃO é bissexto, considera 28/Fev para o cálculo
                        #date(AnoAtual, 2, 28)
                    else
                        // Caso contrário, usa a data de nascimento normal neste ano
                        #date(AnoAtual, MesNascimento, DiaNascimento),
                
                AnosTotais = AnoAtual - AnoNascimento,
                IdadeFinal = if AniversarioEsteAno > DataAtual then AnosTotais - 1 else AnosTotais
            in
                IdadeFinal
in
    CalculaIdade
```

### Período concluídas

```powerquery
let
    DataConclusao = [Primeira vez que entrou na fase Concluído], // Referência à sua coluna existente
    DiaDoMes = if DataConclusao = null then null else Date.Day(DataConclusao),
    
    Periodo = if DiaDoMes = null then null else
              if DiaDoMes >= 1 and DiaDoMes <= 10 then "1 a 10"
              else if DiaDoMes >= 11 and DiaDoMes <= 20 then "11 a 20"
              else if DiaDoMes >= 21 then "21 a 31"
              else null // Fallback
in
    Periodo
```

-----

## 🐍 4. Integração Machine Learning (Python)

Script executado diretamente no Power Query para carregar o modelo (`.pkl`) e gerar previsões no Power BI.

```python
#Script Python no Power Query para previsão da ML 
    import pandas as pd
    import joblib # Para carregar o pipeline salvo
    import os # Para lidar com caminhos de arquivo

    # --- 1. Configurações (AJUSTE AQUI!) ---
    # Caminho COMPLETO para o seu arquivo .pkl salvo.
    # Use o prefixo 'r' para strings brutas ou barras normais '/' para evitar problemas com caminhos do Windows.
    MODEL_PATH = r"coloque o caminho do diretorio seu modelo aqui " # AJUSTE ESTE CAMINHO DO SEU MODELO .PKL

    # Estas listas de colunas DEVEM ser EXATAMENTE as mesmas que você usou no script de TREINAMENTO!
    NUMERICAL_FEATURES = [
        'Idade Cliente',
    ]

    CATEGORICAL_FEATURES = [
        'Período do Mês',
        'Nome do Solicitante',
        'Cliente possui CNH?',
        'Área solicitante',
        'Valor do contrato',
        'Etapa da Proposta',
        'Período concluidas',
    ]

    # --- LISTA DE COLUNAS DE DATA QUE JÁ CHEGAM NO PYTHON EM FORMATO DD/MM/YYYY (Texto) ---
    DD_MM_YYYY_INPUT_COLUMNS = [
        'Primeira vez que entrou na fase Concluído', 
    ]

    # --- LISTA DE OUTRAS COLUNAS DE DATA QUE CHEGAM EM OUTROS FORMATOS (ou como Python datetime) ---
    # São as colunas de data que o Power Query envia ao Python como tipo 'Date'/'DateTime'
    OTHER_DATE_COLUMNS_TO_PROCESS = [
        'Data de Nascimento',
        'Criado em',
    ]

    # Data padrão para preencher NaT no Python antes de formatar para string.
    # Isso garante que o Power Query não receba um 'None' ou 'NaT' que vira null,
    DEFAULT_DATE_PLACEHOLDER_PYTHON = pd.Timestamp('1900-01-01') 

    # --- 2. Carregar o Pipeline Completo ---
    if not os.path.exists(MODEL_PATH):
        raise FileNotFoundError(f"ERRO: O arquivo do modelo '{MODEL_PATH}' não foi encontrado. Verifique o caminho e se o arquivo foi salvo.")

    model_pipeline = joblib.load(MODEL_PATH)

    # --- 3. Preparar Features do Dataset Atual ---
    all_features_expected = NUMERICAL_FEATURES + CATEGORICAL_FEATURES

    # Verifica se as features esperadas existem no dataset do Power BI
    missing_features_in_dataset = [f for f in all_features_expected if f not in dataset.columns]
    if missing_features_in_dataset:
        raise ValueError(f"As seguintes features esperadas pelo modelo não foram encontradas nos dados do Power BI: {missing_features_in_dataset}. Verifique a preparação dos dados no Power Query.")

    df_features = dataset[all_features_expected].copy() 

    # --- 4. Tratar Valores Nulos nas FEATURES (Mesma lógica do treinamento) ---
    # É crucial que o tratamento de nulos nos dados de previsão seja IGUAL ao do treinamento.
    for col in NUMERICAL_FEATURES:
        if df_features[col].isnull().any():
            median_val = df_features[col].median()
            df_features[col] = df_features[col].fillna(median_val)

    for col in CATEGORICAL_FEATURES:
        if df_features[col].isnull().any():
            df_features[col] = df_features[col].fillna('Desconhecido')

    # --- 5. Fazer Previsões Usando o Pipeline ---
    dataset['SLA_Previsto'] = model_pipeline.predict(df_features)
    dataset['Probabilidade_Atraso'] = model_pipeline.predict_proba(df_features)[:, 1]

    # --- 6. TRATAMENTO CRUCIAL: CONVERTER COLUNAS DE DATA PARA TEXTO YYYY-MM-DD ANTES DE RETORNAR ---
    # 6.1. Processar colunas que chegam em formato DD/MM/YYYY (texto)
    for col in DD_MM_YYYY_INPUT_COLUMNS:
        if col in dataset.columns:
            converted_to_datetime = pd.to_datetime(dataset[col], format='%d/%m/%Y', errors='coerce')
            filled_datetime = converted_to_datetime.fillna(DEFAULT_DATE_PLACEHOLDER_PYTHON)
            dataset[col] = filled_datetime.dt.strftime('%Y-%m-%d')

    # 6.2. Processar outras colunas de data (que chegam como datetime ou outro texto, para YYYY-MM-DD)
    for col in OTHER_DATE_COLUMNS_TO_PROCESS:
        if col in dataset.columns:
            converted_to_datetime = pd.to_datetime(dataset[col], errors='coerce')
            filled_datetime = converted_to_datetime.fillna(DEFAULT_DATE_PLACEHOLDER_PYTHON)
            dataset[col] = filled_datetime.dt.strftime('%Y-%m-%d')

    # --- 7. Retornar o DataFrame atualizado ---
    Output = dataset
```
