
```mermaid
sequenceDiagram
    participant Scheduler as Gatilho Digibee
    participant Digibee as Plataforma Digibee
    participant APILG as API LG (SOAP)
    participant APIMindsight as API Mindsight (REST)

    Scheduler->>Digibee: Inicia fluxo de sincronização

    Note over Digibee,APILG: Etapa 1\nObter códigos de empresas
    Digibee->>APILG: ServicoDeEmpresa.ConsultarLista
    APILG-->>Digibee: Lista de empresas

    loop Para cada empresa
        Note over Digibee,APILG: Etapa 2\nIdentificar admissões recentes
        Digibee->>APILG: ServicoDeContratoDeTrabalho.ConsultarListaDeModificados
        Note over Digibee: Filtros:\n- CodigoDaEmpresa\n- PeriodoDeBusca\n- Tipo Inclusão (1)
        APILG-->>Digibee: Matrículas de novos contratos
    end

    Note over Digibee,APILG: Etapa 3\nBuscar detalhes em lotes de 50
    Digibee->>APILG: ServicoDeContratoDeTrabalho.ConsultarListaPorDemanda
    APILG-->>Digibee: Dados contratuais + PessoaId

    Digibee->>APILG: ServicoDeColaborador.ConsultarLista
    APILG-->>Digibee: Dados pessoais (Nome, CPF)

    Digibee->>APILG: ServicoDeContratoDeTrabalho.ConsultarListaDeGestorImediato
    APILG-->>Digibee: Dados do gestor

    Note over Digibee: Etapa 4\nTransformação dos dados
    Digibee->>Digibee: Consolida e transforma em JSON

    Note over Digibee,APIMindsight: Etapa 5\nCadastro na Mindsight
    Digibee->>APIMindsight: POST /employees/create_complete/
    APIMindsight-->>Digibee: Sucesso (201 Created)


    ```
