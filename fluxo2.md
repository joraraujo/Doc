´´´mermaid

sequenceDiagram
    participant Scheduler as Gatilho Digibee
    participant Digibee as Plataforma Digibee
    participant APILG as API LG (SOAP)
    participant APIMindsight as API Mindsight (REST)

    %% Início do Processo Agendado
    Scheduler->>Digibee: Iniciar fluxo de sincronização

    %% Passo 1: Obter a lista de empresas
    Note over Digibee,APILG: Etapa 1: Obter Códigos de todas as empresas
    Digibee->>APILG: 1. Chamada a `ServicoDeEmpresa.ConsultarLista`
    APILG-->>Digibee: 2. Retorna lista de Empresas (ex: Cód 1: BIOLAB, Cód 2: CASA BRANCA)

    %% Passo 2: Iterar e identificar novos colaboradores
    loop Para cada Empresa da lista
        Note over Digibee,APILG: Etapa 2: Identificar admissões recentes para a empresa
        Digibee->>APILG: 3. Chamada a `ServicoDeContratoDeTrabalho.ConsultarListaDeModificados`
        Note over Digibee: Filtros: <br/>- `CodigoDaEmpresa` (do loop) <br/>- `PeriodoDeBusca` <br/>- `TiposDeOperacoes`: Inclusão (1)
        APILG-->>Digibee: 4. Retorna lista de matrículas de novos contratos para essa empresa
    end
    
    %% Passo 3, 4 e 5: Coletar Dados Completos dos Novos Colaboradores
    Note over Digibee,APILG: Etapa 3: Buscar detalhes dos novos colaboradores em lotes de 50
    Digibee->>APILG: 5. Chamada a `ServicoDeContratoDeTrabalho.ConsultarListaPorDemanda`
    Note over Digibee: Usa a lista agregada de matrículas da etapa 2
    APILG-->>Digibee: 6. Retorna dados contratuais e `PessoaId`

    Digibee->>APILG: 7. Chamada a `ServicoDeColaborador.ConsultarLista`
    Note over Digibee: Usa os `PessoaId` da etapa 6
    APILG-->>Digibee: 8. Retorna dados pessoais (Nome, CPF, etc.)

    Digibee->>APILG: 9. Chamada a `ServicoDeContratoDeTrabalho.ConsultarListaDeGestorImediato`
    Note over Digibee: Usa a matrícula e empresa de cada novo contrato
    APILG-->>Digibee: 10. Retorna dados do gestor imediato

    %% Passo 6: Transformação dos Dados
    Note over Digibee: Etapa 4: Preparar dados para a Mindsight
    Digibee->>Digibee: 11. Consolida dados contratuais, pessoais e do gestor
    Digibee->>Digibee: 12. Mapeia e transforma os dados para o formato JSON

    %% Passo 7: Envio para a Mindsight
    Note over Digibee,APIMindsight: Etapa 5: Cadastrar o novo colaborador
    Digibee->>APIMindsight: 13. Chamada `POST /employees/create_complete/`
    Note over Digibee: Payload inclui todos os dados necessários, incluindo `start_date`, e URLs para `area`, `position` e `manager`.
    APIMindsight-->>Digibee: 14. Retorna Sucesso (201 Created) com a URL do novo recurso

    ´´´´
