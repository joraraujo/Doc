
```mermaid

graph TD
    subgraph Orquestração na Digibee
        A[Start: Iniciar Fluxo de Admissão por Agendamento] --> B{1. Buscar Admitidos Recentes};
        B --> C{2. Obter PessoaId de cada contrato};
        C --> D{3. Buscar Dados Pessoais em Lote};
        D --> E{4. Unir Dados e Transformar};
        E --> F[5. Enviar para Mindsight];
    end

    subgraph "Sistema de Origem: LG (SOAP API)"
        B --"Chamar API Contrato de Trabalho"--> B1["ServicoDeContratoDeTrabalho.<br/><b>ConsultarListaDeModificados</b><br/><i>Filtro: TipoDeOperacoes = 1 (Inclusão)</i>"];
        B1 --"Retorna: Lista de Matrículas dos admitidos"--> B;

        C --"Para cada Matrícula"--> C1["ServicoDeContratoDeTrabalho.<br/><b>Consultar</b><br/><i>(ou ConsultarListaPorDemanda)</i>"];
        C1 --"Retorna: Detalhes do contrato, incluindo <b>PessoaId</b>"--> C;

        D --"Agrupar <b>PessoaId</b> em lotes de 50"--> D1["ServicoDeColaborador.<br/><b>ConsultarLista</b>"];
        D1 --"Retorna: Dados pessoais detalhados"--> D;
    end

    subgraph "Sistema de Destino: Mindsight (REST API)"
        F --"Chamar API Mindsight"--> F1["POST /employees/create_complete/"];
        F1 --"Cria o novo colaborador com dados unificados"--> G[End];
    end

    style A fill:#c9f,stroke:#333,stroke-width:2px
    style F1 fill:#d4ffb3,stroke:#333,stroke-width:2px
    style B1 fill:#b3e6ff,stroke:#333,stroke-width:2px
    style C1 fill:#b3e6ff,stroke:#333,stroke-width:2px
    style D1 fill:#b3e6ff,stroke:#333,stroke-width:2px

    ```
