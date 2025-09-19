```mermaid

flowchart TD
    subgraph LG["LG Lugar de Gente (SOAP)"]
        A1[ConsultarListaDeModificados(last_sync)]
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
        C1[POST/PUT /employees]
    end

    %% Ligações
    B1 --> B2 --> B3 --> A1 --> B4 --> B5 --> B6
    B6 -- Ativo --> B7 --> C1 --> B9
    B6 -- Desligado --> B8 --> C1 --> B9
    B9 -- Sucesso --> B10 --> B12
    B9 -- Erro --> B11 --> B12


```
