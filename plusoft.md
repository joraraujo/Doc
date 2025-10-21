```mermaid

graph TD
    A[Início: Sinc_SISNumber] --> B;

    subgraph Inicialização e Configuração
        B{Configuração: PLUSOFT_SincronizaSIS ativo?}
        B -- Sim --> C;
        B -- Não --> FIM;
        C{AjustaCodBarras = True?}
        C -- Sim --> D(Ajusta_CodBarras_Isento_EAI);
        C -- Não --> E;
        D --> E;
    end

    E(ConectaBDSIS: Conecta BD SIS e BD STAGE);

    subgraph Pesquisa de Registros Pendentes
        E --> F["Query em [tb_sis_relato] (BD SIS): Buscar registros onde numeroSequencialFUP is null"];
        F --> G{Há registros para sincronizar?};
        G -- Sim --> H{Para cada Registro codOrigem e codigo};
        G -- Não --> S;
    end

    subgraph Processamento Individual - VerificaControleEAI
        H --> I(Chama VerificaControleEAI);
        I --> J{Tabela PLUSOFT_SIS_Control existe no STAGE?};
        J -- Não --> J_C(Cria Tabela PLUSOFT_SIS_Control e Índices);
        J_C --> K;
        J -- Sim --> K;

        K[Consulta: Verifica Status anterior do codOrigem na PLUSOFT_SIS_Control];
        K --> L{Registro existe E Status = OK?};
        L -- Sim --> M(Finaliza este Registro);
        L -- Não --> N(Chama Plusoft_API_SISUpdate);
    end

    subgraph Chamada da API e Atualização - Plusoft_API_SISUpdate
        N --> O(ExecuteAPIPlusoftSIS: Envia requisição POST p/ PLUSOFT_API_EndPoint);
        O --> P{Resultado da API: OK ou Err};
        P --> Q[Atualiza/Insere: PLUSOFT_SIS_Control (STAGE) com Status e Data APIDateSinc];
        Q --> R[Grava Status/Msg no Log Diário (API_Plusoft_SIS-YYYYMMDD.log)];
        R --> M;
    end

    M --> H;

    S(ConectaBDSIS(false): Desconecta BD SIS e BD STAGE);
    S --> FIM(Fim da Rotina);



```
