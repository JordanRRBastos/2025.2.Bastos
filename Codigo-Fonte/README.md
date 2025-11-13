 Abaixo estão todos os codigos das medidas, colunas calculadas, transformações da Power Query para a criação dos dashboards e o script Python utilizado para trazer o modelo de Machine Learning para dentro do Power BI.

⚙️ Medidas (DAX)
🟩 Concluídos < 3 dias
Concluídos < 3 dias =
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    FILTER(
        'FAT_CERTIFICADOS',
        'FAT_CERTIFICADOS'[Fase atual] = "Concluído"
            && 'Medidas'[Média Lead Time Total (Emitidos)] <= 3
    )
)

🟩 % Concluídos < 3 dias
% Concluídos < 3 dias =
DIVIDE(
    [Concluídos < 3 dias],
    [Total Solicitações Concluídas],
    0
)

🟩 Média Lead Time Agendamento (Arquivados)
Média Lead Time Agendamento (Arquivados) =
 AVERAGEX(
    FAT_CERTIFICADOS,
    DATEDIFF(FAT_CERTIFICADOS[Primeira vez que entrou na fase Agendamento], FAT_CERTIFICADOS[Primeira vez que entrou na fase Arquivado], DAY)
 )

🟩 Média Lead Time Agendamento (Emitidos)
Média Lead Time Agendamento (Emitidos) =
 AVERAGEX(
    FAT_CERTIFICADOS,
    DATEDIFF(FAT_CERTIFICADOS[Ultima vez que entrou na fase Agendamento], FAT_CERTIFICADOS[Primeira vez que entrou na fase Concluído], DAY)
 )

🟩 Média Lead Time Total (Arquivados)
Média Lead Time Total (Arquivados) =
 AVERAGEX(
    FAT_CERTIFICADOS,
    DATEDIFF(FAT_CERTIFICADOS[Criado em], FAT_CERTIFICADOS[Primeira vez que entrou na fase Arquivado], DAY)
 )

🟩 Média Lead Time Total (Emitidos)
Média Lead Time Total (Emitidos) =
 AVERAGEX(
    FAT_CERTIFICADOS,
    DATEDIFF(FAT_CERTIFICADOS[Criado em], FAT_CERTIFICADOS[Primeira vez que entrou na fase Concluído], DAY)
 )

🟩 Tickets Ativos Contagem (Somente 1s)
Tickets Ativos Contagem (Somente 1s) =
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    'FAT_CERTIFICADOS'[solicitacao ativa] = 1
)

🟩 Tickets Atrasados Vencidos (Contagem)
Tickets Atrasados Vencidos (Contagem) =
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    'FAT_CERTIFICADOS'[solicitacao ativa] = 1,
    'FAT_CERTIFICADOS'[Status_Risco_SLA_Coluna] = "Atrasado Vencido (Real)"
)

🟩 Tickets em Risco Alto
Tickets em Risco Alto =
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    'FAT_CERTIFICADOS'[solicitacao ativa] = 1, // Filtra apenas tickets ativos
    'FAT_CERTIFICADOS'[SLA_Previsto] = 1 // Previsto para atrasar (ou já atrasado)
)

🟩 Tickets em Risco Alto (Contagem)
Tickets em Risco Alto (Contagem) =
CALCULATE(
    COUNTROWS('FAT_CERTIFICADOS'),
    'FAT_CERTIFICADOS'[solicitacao ativa] = 1,
    'FAT_CERTIFICADOS'[Status_Risco_SLA_Coluna] = "Alto Risco (Previsto)"
)

🟩 Total Solicitações
Total Solicitações = CALCULATE(
    COUNT('FAT_CERTIFICADOS'[Código]),
           USERELATIONSHIP(DIM_CALENDARIO[Data],'FAT_CERTIFICADOS'[Criado em]))

🟩 Total Solicitações Arquivadas
Total Solicitações Arquivadas = CALCULATE(
    COUNT('FAT_CERTIFICADOS'[Código]),
    'FAT_CERTIFICADOS'[Fase atual] = "Arquivado",
           USERELATIONSHIP(DIM_CALENDARIO[Data],'FAT_CERTIFICADOS'[Primeira vez que entrou na fase Arquivado]))

🟩 Total Solicitações Concluídas
Total Solicitações Concluídas = CALCULATE(
    COUNT('FAT_CERTIFICADOS'[Código]),
    'FAT_CERTIFICADOS'[Fase atual] = "Concluído",
           USERELATIONSHIP(DIM_CALENDARIO[Data],'FAT_CERTIFICADOS'[Primeira vez que entrou na fase Concluído]))

🟩 Total Solicitações em Agendamento
Total Solicitações em Agendamento = CALCULATE(
    COUNT('FAT_CERTIFICADOS'[Código]),
    'FAT_CERTIFICADOS'[Fase atual] = "Agendamento"
)

📘 Colunas Calculadas
SLA_Dias_Uteis
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

SLA_Atrasado_3Dias_Corrigido_Coluna
SLA_Atrasado_3Dias_Corrigido_Coluna =
VAR RawValue = 'FAT_CERTIFICADOS'[SLA_Atrasado_3Dias]
RETURN
    IF(
        RawValue = 10,
        1,
        RawValue
    )

Probabilidade_Atraso_Final_Coluna
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

Idade Cliente Corrigida_Coluna
Idade Cliente Corrigida_Coluna =
DIVIDE('FAT_CERTIFICADOS'[Idade Cliente], 10)

Status_Risco_SLA_Coluna
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

⚡ Colunas Adicionadas na Power Query
SLA_Atrasado_3Dias
SLA_Atrasado_3Dias = let
    DataDeEntrada = [Criado em],
    DataDeConclusao = [Última vez que entrou na fase Concluído],
    LimiteSLADias = 3,

    SLA_Status_Original =
        if DataDeEntrada = null or DataDeConclusao = null then null
        else
            let
                DiasGastos = Duration.Days(DataDeConclusao - DataDeEntrada)
            in
                if DiasGastos > LimiteSLADias then 1 else 0
in
    SLA_Status_Original

Idade Cliente
Idade Cliente = let
    DataNascimentoRaw = [Data de Nascimento],
    DataAtual = Date.From(DateTime.LocalNow()),

    DataNascimento = try Date.From(DataNascimentoRaw) otherwise null,

    CalculaIdade =
        if DataNascimento = null or DataNascimento > DataAtual then
            null
        else
            let
                AnoNascimento = Date.Year(DataNascimento),
                MesNascimento = Date.Month(DataNascimento),
                DiaNascimento = Date.Day(DataNascimento),

                AnoAtual = Date.Year(DataAtual),

                AniversarioEsteAno =
                    if MesNascimento = 2 and DiaNascimento = 29 and Date.DaysInMonth(#date(AnoAtual, 2, 1)) = 28 then
                        #date(AnoAtual, 2, 28)
                    else
                        #date(AnoAtual, MesNascimento, DiaNascimento),

                AnosTotais = AnoAtual - AnoNascimento,
                IdadeFinal = if AniversarioEsteAno > DataAtual then AnosTotais - 1 else AnosTotais
            in
                IdadeFinal
in
    CalculaIdade

Período concluídas
Período concluídas = let
    DataConclusao = [Primeira vez que entrou na fase Concluído],
    DiaDoMes = if DataConclusao = null then null else Date.Day(DataConclusao),

    Periodo = if DiaDoMes = null then null else
              if DiaDoMes >= 1 and DiaDoMes <= 10 then "1 a 10"
              else if DiaDoMes >= 11 and DiaDoMes <= 20 then "11 a 20"
              else if DiaDoMes >= 21 then "21 a 31"
              else null
in
    Período

🧠 Script Python – Integração do Modelo de Machine Learning no Power BI
# Script Python no Power Query para previsão da ML
import pandas as pd
import joblib
import os

# --- 1. Configurações (AJUSTE AQUI!) ---
MODEL_PATH = r"coloque o caminho do diretorio seu modelo aqui "

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

DD_MM_YYYY_INPUT_COLUMNS = [
    'Primeira vez que entrou na fase Concluído',
]

OTHER_DATE_COLUMNS_TO_PROCESS = [
    'Data de Nascimento',
    'Criado em',
]

DEFAULT_DATE_PLACEHOLDER_PYTHON = pd.Timestamp('1900-01-01')

# --- 2. Carregar o Pipeline Completo ---
if not os.path.exists(MODEL_PATH):
    raise FileNotFoundError(f"ERRO: O arquivo do modelo '{MODEL_PATH}' não foi encontrado.")

model_pipeline = joblib.load(MODEL_PATH)

# --- 3. Preparar Features do Dataset Atual ---
all_features_expected = NUMERICAL_FEATURES + CATEGORICAL_FEATURES
missing_features_in_dataset = [f for f in all_features_expected if f not in dataset.columns]

if missing_features_in_dataset:
    raise ValueError(f"As seguintes features esperadas pelo modelo não foram encontradas: {missing_features_in_dataset}.")

df_features = dataset[all_features_expected].copy()

# --- 4. Tratar Valores Nulos ---
for col in NUMERICAL_FEATURES:
    if df_features[col].isnull().any():
        median_val = df_features[col].median()
        df_features[col] = df_features[col].fillna(median_val)

for col in CATEGORICAL_FEATURES:
    if df_features[col].isnull().any():
        df_features[col] = df_features[col].fillna('Desconhecido')

# --- 5. Fazer Previsões ---
dataset['SLA_Previsto'] = model_pipeline.predict(df_features)
dataset['Probabilidade_Atraso'] = model_pipeline.predict_proba(df_features)[:, 1]

# --- 6. Converter Colunas de Data ---
for col in DD_MM_YYYY_INPUT_COLUMNS:
    if col in dataset.columns:
        converted_to_datetime = pd.to_datetime(dataset[col], format='%d/%m/%Y', errors='coerce')
        filled_datetime = converted_to_datetime.fillna(DEFAULT_DATE_PLACEHOLDER_PYTHON)
        dataset[col] = filled_datetime.dt.strftime('%Y-%m-%d')

for col in OTHER_DATE_COLUMNS_TO_PROCESS:
    if col in dataset.columns:
        converted_to_datetime = pd.to_datetime(dataset[col], errors='coerce')
        filled_datetime = converted_to_datetime.fillna(DEFAULT_DATE_PLACEHOLDER_PYTHON)
        dataset[col] = filled_datetime.dt.strftime('%Y-%m-%d')

# --- 7. Retornar o DataFrame atualizado ---
Output = dataset
