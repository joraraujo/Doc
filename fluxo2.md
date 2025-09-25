
```mermaid

sequenceDiagram
    participant Scheduler as Gatilho Digibee
    participant Digibee as Plataforma Digibee
    participant APILG as API LG - SOAP
    participant APIMindsight as API Mindsight - REST

    Scheduler->>Digibee: Inicia fluxo de sincronizacao

    note over Digibee,APILG: Etapa 1 - Obter codigos das empresas
    Digibee->>APILG: ServicoDeEmpresa.ConsultarLista
    APILG->>Digibee: Lista de empresas

    loop Para cada empresa
        note over Digibee,APILG: Etapa 2 - Identificar admissoes recentes
        Digibee->>APILG: ServicoDeContratoDeTrabalho.ConsultarListaDeModificados
        note over Digibee: Filtros: CodigoDaEmpresa; PeriodoDeBusca; TipoInclusao=1
        APILG->>Digibee: Matriculas de novos contratos
    end

    note over Digibee,APILG: Etapa 3 - Buscar detalhes em lotes de 50
    Digibee->>APILG: ServicoDeContratoDeTrabalho.ConsultarListaPorDemanda
    APILG->>Digibee: Dados contratuais + PessoaId

    Digibee->>APILG: ServicoDeColaborador.ConsultarLista
    APILG->>Digibee: Dados pessoais (Nome, CPF)

    Digibee->>APILG: ServicoDeContratoDeTrabalho.ConsultarListaDeGestorImediato
    APILG->>Digibee: Dados do gestor

    note over Digibee: Etapa 4 - Transformacao e mapeamento
    Digibee->>Digibee: Consolida dados e transforma em JSON

    note over Digibee,APIMindsight: Etapa 5 - Cadastro na Mindsight
    Digibee->>APIMindsight: POST /employees/create_complete/
    APIMindsight->>Digibee: Sucesso 201 Created

    ```
