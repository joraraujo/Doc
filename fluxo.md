```mermaid


flowchart TD
    subgraph LG["LG Lugar de Gente (SOAP)"]
        A1[ConsultarListaDeModificados]
    end

    subgraph Digibee["Digibee iPaaS"]
        B1[Agendamento Timer]
        B2[Obter last_sync do Datastore]
        B3[Chamar SOAP LG]
        B4[Transformar dados LG → JSON Mindsight]
        B5[Para cada colaborador]
        B6{Status ativo ou desligado?}
        B7[Enviar status ACTIVE → Mindsight API]
        B8[Enviar status INACTIVE → Mindsight API]
        B9[Verificar resposta da API]
        B10[Log sucesso]
        B11[Registrar erro + enviar para retry]
        B12[Atualizar last_sync no Datastore]
    end

    subgraph Mindsight["Mindsight (REST API)"]
        C1[POST ou PUT /employees]
    end

    %% Ligações
    B1 --> B2 --> B3 --> A1 --> B4 --> B5 --> B6
    B6 -- Ativo --> B7 --> C1 --> B9
    B6 -- Desligado --> B8 --> C1 --> B9
    B9 -- Sucesso --> B10 --> B12
    B9 -- Erro --> B11 --> B12

```


```mermaid
graph TD
    subgraph Sincronizacao Continua
        direction TB

        A2[Inicio da Sincronizacao Incremental] --> B2[1. Recuperar Data/Hora da Ultima Sincronizacao do estado];
        B2 --> C2[2. Chamar ConsultarListaDeModificados no ServicoDeContratoDeTrabalho da LG\nPeriodo de Busca: UltimaSincronizacao -> Agora];
        C2 --> D2{Existem registros modificados?};
        D2 -- Nao --> E2[Fim da Sincronizacao - Sem alteracoes];
        D2 -- Sim --> F2[3. Obter lista de Matriculas e tipo de operacao - Inclusao, Alteracao, Exclusao];
        
        F2 --> G2{Loop para cada Matricula Modificada};
        G2 --> H2[4. Consultar dados completos do contrato - ServicoDeContratoDeTrabalho];
        H2 --> I2[5. Consultar dados pessoais completos - ServicoDeColaborador];
        I2 --> J2[6. Combinar e Traduzir dados];
        J2 --> K2{Qual o tipo de operacao?};

        K2 -- Inclusao/Alteracao --> L2[7a. Verificar existencia na Mindsight\nGET /employees/?employee_code={matricula}];
        L2 -- Existe --> M2[8a. Atualizar Funcionario\nPreparar payload e chamar PUT /employees/{id}/];
        L2 -- Nao Existe --> M3[8b. Criar Funcionario\nPreparar payload e chamar POST /employees/create_complete/];
        
        K2 -- Exclusao Rescisao --> L3[7b. Desativar Funcionario\nBuscar URL em _actions.deactivate];
        L3 -- URL Valida --> M4[8c. POST para a URL de desativacao\ncom data e motivo da rescisao];
        
        subgraph Pos-Operacao Mindsight
            M2 --> N2{Sucesso?};
            M3 --> N2;
            M4 --> N2;
            N2 -- Sim --> O2[9. Logar sucesso];
            N2 -- Nao --> O3[9. Logar erro e mover para Fila de Retentativas];
            O2 --> P2{Fim do Loop?};
            O3 --> P2;
        end

        P2 -- Nao --> G2;
        P2 -- Sim --> Q2[10. Atualizar Data/Hora da Ultima Sincronizacao\ncom o horario atual];
        Q2 --> R2[Fim da Sincronizacao Incremental];
    end

```
