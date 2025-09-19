Com base na documentação fornecida, aqui está um procedimento detalhado para sincronizar os dados do sistema LG (Lugar de Gente) com a Mindsight, onde a Mindsight é o sistema que **recebe** os dados.

Este procedimento é otimizado para ser eficiente, buscando apenas os registros modificados desde a última sincronização.

---

### **Procedimento de Sincronização LG -> Mindsight**

#### **1. Configurações Iniciais**

*   **Token de Autenticação da Mindsight:** Obtenha um token de acesso válido para a API da Mindsight. Este token deve ser incluído no cabeçalho `Authorization` de todas as requisições.
    *   Exemplo: `Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b`
*   **Credenciais da API SOAP do LG:** Configure as credenciais (`Usuario`, `Senha`, `GuidTenant`) e o ambiente (`Ambiente`) para autenticar as requisições SOAP.
*   **URLs Base:**
    *   LG SOAP: `https://<ambiente>-api1.lg.com.br/v2/ServicoDeContratoDeTrabalho`
    *   Mindsight REST: `https://<servidor>/<empresa>/api/v1/`
*   **Data da Última Sincronização:** Armazene localmente a data e hora da última execução bem-sucedida deste procedimento. Esta será a referência para buscar apenas registros modificados.

---

#### **2. Fluxo Principal de Sincronização**

O fluxo abaixo deve ser executado periodicamente (por exemplo, a cada 30 minutos ou diariamente).

##### **Passo 2.1: Identificar Contratos Modificados no LG**

Utilize o serviço `ConsultarListaDeModificados` da API SOAP do LG para obter uma lista de matrículas cujos contratos foram alterados desde a última sincronização.

*   **Endpoint SOAP:** `ConsultarListaDeModificados`
*   **Parâmetros:**
    *   `CodigoDaEmpresa`: O código da empresa no LG.
    *   `PeriodoDeBusca.DataInicio`: A data da última sincronização.
    *   `PeriodoDeBusca.DataFim`: A data/hora atual.
    *   `TiposDeOperacoes`: Inclua `1 (Inclusão)`, `2 (Alteração)` e `3 (Exclusão)` para capturar todas as mudanças.
    *   `TiposDeSituacao`: Inclua todos os tipos (1 a 5) para não filtrar por situação.
*   **Saída:** Uma lista de objetos `ContratoDeTrabalhoModificado`, contendo `Matricula` e `CodigoDaEmpresa`.

##### **Passo 2.2: Para Cada Matrícula Modificada, Obter Dados Completos**

Para cada matrícula identificada no Passo 2.1, faça as seguintes chamadas:

1.  **Obter Dados Contratuais:**
    *   **Endpoint SOAP:** `Consultar`
    *   **Parâmetros:** `FiltroComIdentificacaoDeContrato` com `CodigoDaEmpresa` e `Matricula`.
    *   **Saída:** Um objeto `ContratoDeTrabalhoV3` contendo todos os dados contratuais, incluindo `Alocacao` (cargo, área, gestor) e `DadosGerais` (data de admissão, situação).

2.  **Obter Dados Pessoais:**
    *   Extraia o `PessoaId` do objeto `ContratoDeTrabalhoV3.Pessoa.PessoaId`.
    *   **Endpoint SOAP:** `ConsultarLista` (Serviço de Colaborador)
    *   **Parâmetros:** Uma lista com o `PessoaId` obtido. (Você pode acumular até 50 `PessoaId` para otimizar as chamadas).
    *   **Saída:** Um objeto `ColaboradorV3` contendo nome, CPF, data de nascimento, etc.

##### **Passo 2.3: Mapear Dados do LG para o Formato da Mindsight**

Com os dados contratuais (`ContratoDeTrabalhoV3`) e pessoais (`ColaboradorV3`) em mãos, mapeie-os para o formato esperado pela API da Mindsight.

| Campo na Mindsight (Endpoint `create_complete` ou `activate`) | Fonte no LG | Observações |
| :--- | :--- | :--- |
| `first_name` | `Pessoa.DadosPessoaisFichaV2.Nome` (ou dividir por espaço) | |
| `last_name` | `Pessoa.DadosPessoaisFichaV2.Nome` (ou dividir por espaço) | |
| `email` | `Pessoa.Contato.EmailCorporativo` | |
| `employee_code` | `Matricula` | |
| `start_date` | `DadosGeraisDoContratoV2.DataAdmissao` | |
| `cpf` | `Pessoa.DadosPessoaisFichaV2.CpfCnpj` | |
| `birth_date` | `Pessoa.DadosPessoaisFichaV2.DataDeNascimento` | |
| `area` | `AlocacaoDoContratoV2.UnidadeOrganizacional` | *Precisa ser a URL do recurso na Mindsight.* |
| `position` | `AlocacaoDoContratoV2.CargoEFuncaoEfetivo.Cargo` | *Precisa ser a URL do recurso na Mindsight.* |
| `manager` | `AlocacaoDoContratoV2.ListaDeGestores[0]` | *Precisa ser a URL do recurso na Mindsight.* |

##### **Passo 2.4: Sincronizar Áreas, Cargos e Gestores na Mindsight**

Antes de criar ou atualizar um funcionário, é crucial garantir que as Áreas, Cargos e Gestores referenciados já existam na Mindsight.

1.  **Sincronizar Área:**
    *   Verifique se a área (Unidade Organizacional do LG) já existe na Mindsight consultando `/api/v1/areas/?code=<codigo_da_area>`.
    *   Se não existir, crie-a com `POST /api/v1/areas/create_complete/`.
    *   Armazene localmente o mapeamento entre o código da área no LG e a URL do recurso na Mindsight.

2.  **Sincronizar Cargo:**
    *   Verifique se o cargo já existe na Mindsight consultando `/api/v1/positions/?code=<codigo_do_cargo>`.
    *   Se não existir, crie-o com `POST /api/v1/positions/create_complete/`.
    *   Armazene localmente o mapeamento entre o código do cargo no LG e a URL do recurso na Mindsight.

3.  **Sincronizar Gestor:**
    *   Verifique se o gestor (outro funcionário) já existe na Mindsight consultando `/api/v1/employees/?employee_code=<matricula_do_gestor>`.
    *   Se não existir, você pode optar por:
        *   Criá-lo primeiro (se ele também estiver na lista de modificados, será processado em seguida).
        *   Deixar o campo `manager` vazio na criação/atualização do funcionário subordinado.

##### **Passo 2.5: Criar ou Atualizar Funcionário na Mindsight**

Com todas as dependências (área, cargo, gestor) resolvidas, envie os dados do funcionário para a Mindsight.

*   **Se o funcionário NÃO existe na Mindsight:**
    *   Use o endpoint `POST /api/v1/employees/create_complete/` para criá-lo.
    *   Exemplo de Payload:
        ```json
        {
          "first_name": "João",
          "last_name": "Silva",
          "email": "joao.silva@empresa.com",
          "employee_code": "FUNC001",
          "start_date": "2024-05-01",
          "cpf": "123.456.789-00",
          "birth_date": "1990-01-01",
          "area": "https://<servidor>/<empresa>/api/v1/areas/5/",
          "position": "https://<servidor>/<empresa>/api/v1/positions/3/",
          "manager": "https://<servidor>/<empresa>/api/v1/employees/123/"
        }
        ```

*   **Se o funcionário JÁ existe e foi DEMITIDO (Situação 5 - Rescisão):**
    1.  Busque o funcionário para obter a URL de desativação:
        ```bash
        GET /api/v1/employees/1/
        ```
        Procure o campo: `"_actions.deactivate": "https://<servidor>/<empresa>/api/v1/employees/1/deactivate/"`
    2.  Envie a requisição POST para a URL obtida:
        ```json
        {
          "end_date": "2024-05-31",
          "termination_type": "resigned",
          "termination_reason": "Pedido de demissão"
        }
        ```

*   **Se o funcionário foi REATIVADO (ex: voltou de uma rescisão):**
    1.  Busque o funcionário para obter a URL de reativação (`_actions.activate`).
    2.  Envie a requisição POST com os novos dados (área, cargo, etc.).

##### **Passo 2.6: Atualizar Data da Última Sincronização**

Após processar com sucesso todos os registros modificados, atualize a data da "última sincronização" para a data/hora em que esta execução foi iniciada. Isso garantirá que a próxima execução busque apenas as alterações ocorridas a partir deste ponto.

---

#### **3. Tratamento de Erros e Retentativas**

*   Implemente um mecanismo robusto de tratamento de erros.
*   Para erros transitórios (ex: timeout, erro 5xx), implemente uma estratégia de retentativa com backoff exponencial.
*   Registre todos os erros em um log para análise posterior.

---

#### **4. Otimizações**

*   **Processamento em Lote:** Agrupe as chamadas SOAP sempre que possível (ex: `ConsultarLista` para colaboradores aceita até 50 `PessoaId`).
*   **Cache Local:** Mantenha um cache local dos mapeamentos (código LG -> URL Mindsight) para áreas, cargos e gestores, evitando consultas desnecessárias à API da Mindsight.
*   **Paginação:** Se a lista de modificados for muito grande, implemente paginação no serviço `ConsultarListaDeModificados` (embora a documentação não mencione paginação explícita para este endpoint, é uma boa prática estar preparado).

