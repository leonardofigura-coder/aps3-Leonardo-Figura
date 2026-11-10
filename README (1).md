# Assistente de IA para Exportação — Análise Regulatória da Exportação

Documentação técnica (Diagrams as Code) da funcionalidade de **análise regulatória da exportação**.

**Requisitos relacionados**

| Requisito | Descrição (resumo) | Artefatos que o cobrem |
|-----------|--------------------|------------------------|
| RF05 | Consulta e classificação de NCM no Siscomex | 1, 2, 3, 4 |
| RF07 | Análise dos documentos exigidos por país de destino | 1, 2, 3, 4 |
| RF08 | Cálculo de frete com base em agentes de carga | 1, 2, 3, 4 |
| RF09 | Atualização da base normativa pelo curador regulatório | 2, 3, 4, 5 |

> Os diagramas Mermaid renderizam direto no GitHub/GitLab. Os diagramas PlantUML (Artefatos 2) podem ser renderizados em https://www.plantuml.com/plantuml ou pela extensão PlantUML do VS Code.

---

## Estrutura dos artefatos exigidos

A entrega contém obrigatoriamente **5 artefatos técnicos**, codificados em texto declarativo e incorporados neste `README.md`:

| # | Artefato | Ferramenta | Seção |
|---|----------|-----------|-------|
| 1 | Diagrama de Sequência (UML / DaC) | Mermaid | [Artefato 1](#-artefato-1--diagrama-de-sequência-mermaid) |
| 2 | Diagrama de Atividades com Raias (UML / DaC) | PlantUML | [Artefato 2](#-artefato-2--diagrama-de-atividades-com-raias-plantuml) |
| 3 | Diagrama de Transição de Estados (State Machine / DaC) | Mermaid | [Artefato 3](#-artefato-3--diagrama-de-transição-de-estados-mermaid) |
| 4 | Visão Arquitetural Macro C4 (Models-as-Code) | Mermaid | [Artefato 4](#-artefato-4--visão-arquitetural-c4-nível-2-contêineres-mermaid) |
| 5 | Registro de Decisão Arquitetural (ADR 0001) | Markdown (Nygard) | [Artefato 5](#-artefato-5--registro-de-decisão-arquitetural) |

---

## 📄 Artefato 1 — Diagrama de Sequência (Mermaid)

**Objetivo:** mapear a troca cronológica de mensagens entre instâncias em tempo de execução.

**Requisitos técnicos e como foram atendidos**

- ✅ Sintaxe Mermaid (`sequenceDiagram`).
- ✅ Chamadas síncronas bloqueantes com `->>` e chamadas assíncronas/mensageria com `-)` (a seta que o enunciado escreve como `-\>`; no Mermaid a assíncrona é `-)`).
- ✅ Fragmentos combinados: `alt/else` (tratamento de NCM inválida) e `loop` (iteração por país de destino).
- ✅ Retorno de dados com DTOs (`-->>` com `NcmDTO`, `ClassificacaoDTO`, `ListaDocumentosDTO`, `RelatorioExportacaoDTO` etc.).
- ✅ Regra de consistência: toda mensagem para uma linha de vida B corresponde a uma operação pública de B, conforme a tabela de operações abaixo.

Fluxo principal: o usuário solicita a análise, o sistema classifica a NCM no Siscomex, levanta os documentos de cada país de destino, cota o frete e publica eventos de forma assíncrona.

- `->>` : chamada síncrona bloqueante
- `-)` : chamada assíncrona (mensageria)
- `-->>` : retorno de dados (DTO)

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuário de Exportação
    participant API as ExportacaoController
    participant CLS as ClassificacaoService
    participant SIS as SiscomexGateway
    participant DOC as AnaliseDocumentalService
    participant NOR as BaseNormativaRepository
    participant FRT as FreteService
    participant MQ as BarramentoEventos
    participant NTF as NotificadorCurador

    U ->> API: solicitarAnalise(SolicitacaoExportacaoDTO)
    API ->> CLS: classificarNcm(ConsultaNcmDTO)
    CLS ->> SIS: consultarNcm(codigoNcm)
    SIS -->> CLS: NcmDTO

    alt ncmValida == true
        CLS -->> API: ClassificacaoDTO
        loop para cada país de destino
            API ->> DOC: levantarDocumentos(ClassificacaoDTO, paisDestino)
            DOC ->> NOR: buscarRequisitos(codigoNcm, paisDestino)
            NOR -->> DOC: RequisitoNormativoDTO
            DOC -->> API: ListaDocumentosDTO
        end
        API ->> FRT: cotarFrete(ConsultaFreteDTO)
        FRT -->> API: CotacaoFreteDTO
        API -) MQ: publicarEvento(EventoDominioDTO)
        API -->> U: RelatorioExportacaoDTO
    else ncmValida == false
        CLS -->> API: NcmInvalidaErro
        API -) MQ: publicarEvento(EventoDominioDTO)
        MQ -) NTF: notificarCurador(EventoDominioDTO)
        API -->> U: ErroValidacaoDTO
    end
```

### Regra de consistência — operações públicas por classe/interface

Toda mensagem recebida por uma linha de vida corresponde a uma operação pública declarada abaixo.

| Classe / Interface (linha de vida) | Operação pública | Mensagem no diagrama |
|---|---|---|
| `ExportacaoController` | `solicitarAnalise(SolicitacaoExportacaoDTO): RelatorioExportacaoDTO` | Usuário → API |
| `ClassificacaoService` | `classificarNcm(ConsultaNcmDTO): ClassificacaoDTO` | API → CLS |
| `SiscomexGateway` | `consultarNcm(String codigoNcm): NcmDTO` | CLS → SIS |
| `AnaliseDocumentalService` | `levantarDocumentos(ClassificacaoDTO, String paisDestino): ListaDocumentosDTO` | API → DOC |
| `BaseNormativaRepository` | `buscarRequisitos(String codigoNcm, String paisDestino): RequisitoNormativoDTO` | DOC → NOR |
| `FreteService` | `cotarFrete(ConsultaFreteDTO): CotacaoFreteDTO` | API → FRT |
| `BarramentoEventos` | `publicarEvento(EventoDominioDTO): void` | API → MQ |
| `NotificadorCurador` | `notificarCurador(EventoDominioDTO): void` | MQ → NTF |

### DTOs utilizados

| DTO | Campos principais |
|---|---|
| `SolicitacaoExportacaoDTO` | descricaoProduto, paisesDestino[], pesoKg, volumeM3 |
| `ConsultaNcmDTO` | descricaoProduto, codigoNcmInformado (opcional) |
| `NcmDTO` | codigoNcm, descricaoOficial, ncmValida |
| `ClassificacaoDTO` | codigoNcm, descricaoOficial, ncmValida |
| `RequisitoNormativoDTO` | codigoNcm, paisDestino, normas[], versaoBase |
| `ListaDocumentosDTO` | paisDestino, documentos[], observacoes |
| `ConsultaFreteDTO` | paisDestino, pesoKg, volumeM3 |
| `CotacaoFreteDTO` | agente, valor, moeda, prazoDias |
| `EventoDominioDTO` | tipo, idAnalise, dataHora, payload |
| `RelatorioExportacaoDTO` | classificacao, documentosPorPais[], cotacoes[] |
| `ErroValidacaoDTO` | codigo, mensagem, campo |

---

## 📄 Artefato 2 — Diagrama de Atividades com Raias (PlantUML)

**Objetivo:** mapear o fluxo de trabalho (workflow) e a concorrência do processo de negócio.

**Requisitos técnicos e como foram atendidos**

- ✅ Sintaxe PlantUML (o Mermaid não possui raias nativas).
- ✅ 4 raias de partição (mínimo exigido: 3): Usuário de Exportação, Sistema Back-End, Serviço Externo e Curador Regulatório.
- ✅ Nós de decisão com condições de guarda mutuamente exclusivas: `[ncmValida == true]` × `[ncmValida == false]` e `[baseVigente == true]` × `[baseVigente == false]`.
- ✅ Execução paralela com Barra de Fork (`fork` / `fork again`) e Barra de Join (`end fork`): levantamento de documentos (RF07), cálculo de frete (RF08) e verificação da base normativa (RF09).

O Mermaid não possui raias nativas; por isso este artefato usa PlantUML. São 4 raias, decisões com guardas mutuamente exclusivas e execução paralela com **Fork** e **Join** (`fork` … `end fork`).

```plantuml
@startuml
title Fluxo de Análise Regulatória da Exportação

|Usuário de Exportação|
start
:Informar produto, países de destino e carga;

|Sistema Back-End|
:Validar campos obrigatórios da solicitação;
:Solicitar classificação NCM (RF05);

|Serviço Externo|
:Siscomex retorna os dados da NCM;

|Sistema Back-End|
if (NCM reconhecida?) then ([ncmValida == true])
  fork
    :Levantar documentos exigidos por país (RF07);
  fork again
    |Serviço Externo|
    :Agentes de carga retornam tarifas (RF08);
    |Sistema Back-End|
    :Calcular frete estimado (RF08);
  fork again
    :Verificar vigência da base normativa (RF09);
  end fork

  if (Base normativa vigente?) then ([baseVigente == true])
    :Manter requisitos levantados;
  else ([baseVigente == false])
    |Curador Regulatório|
    :Revisar e publicar atualização normativa (RF09);
    |Sistema Back-End|
    :Reprocessar requisitos documentais (RF07);
  endif

  :Consolidar relatório de exportação;
  |Usuário de Exportação|
  :Receber relatório com NCM, documentos e frete;
  stop
else ([ncmValida == false])
  |Sistema Back-End|
  :Gerar erro de validação e notificar curador;
  |Usuário de Exportação|
  :Receber erro e corrigir a descrição do produto;
  stop
endif
@enduml
```

---

## 📄 Artefato 3 — Diagrama de Transição de Estados (Mermaid)

**Objetivo:** mapear o ciclo de vida rigoroso da entidade de domínio principal (`AnaliseExportacao`).

**Requisitos técnicos e como foram atendidos**

- ✅ Sintaxe Mermaid `stateDiagram-v2`.
- ✅ Estado inicial (`[*] --> Rascunho`), subestados (`EmClassificacao` e `EmAnaliseRegulatoria`, este com regiões concorrentes) e três estados terminais (`Rejeitada`, `Arquivada`, `Cancelada`).
- ✅ Sintaxe formal `Gatilho [CondiçãoDeGuarda] / AçãoDeEfeito()` em várias setas, por exemplo `submeter [dadosCompletos == true] / validarEntrada()` e `prazoExpirado [diasDesdeConclusao > 90] / arquivar()`.
- ✅ Ações internas nos estados `ConsultandoSiscomex` (`entry /`, `do /`, `exit /`) e `LevantandoDocumentos` (`entry /`, `do /`).

Entidade principal: **`AnaliseExportacao`**. Contém estado inicial, subestados, regiões concorrentes, estados terminais, transições no formato `Gatilho [Guarda] / Ação()` e compartimentos internos `entry /`, `do /` e `exit /`.

```mermaid
stateDiagram-v2
    [*] --> Rascunho

    Rascunho --> EmClassificacao : submeter [dadosCompletos == true] / validarEntrada()

    state EmClassificacao {
        [*] --> ConsultandoSiscomex
        ConsultandoSiscomex --> NcmConfirmada : respostaSiscomex [ncmValida == true] / registrarNcm()
    }

    EmClassificacao --> Rejeitada : respostaSiscomex [ncmValida == false] / notificarUsuario()
    EmClassificacao --> EmAnaliseRegulatoria : ncmConfirmada / iniciarAnalise()

    state EmAnaliseRegulatoria {
        [*] --> LevantandoDocumentos
        LevantandoDocumentos --> DocumentosProntos : requisitosEncontrados / salvarDocumentos()
        --
        [*] --> CotandoFrete
        CotandoFrete --> FreteCotado : tarifasRecebidas [qtdTarifas > 0] / calcularFrete()
    }

    EmAnaliseRegulatoria --> Concluida : analiseFinalizada / gerarRelatorio()
    Concluida --> EmAnaliseRegulatoria : normaAtualizada [versaoBase > versaoAnalise] / reabrirAnalise()
    Concluida --> Arquivada : prazoExpirado [diasDesdeConclusao > 90] / arquivar()

    Rascunho --> Cancelada : cancelar / descartarRascunho()

    Rejeitada --> [*]
    Arquivada --> [*]
    Cancelada --> [*]

    ConsultandoSiscomex : entry / abrirSessaoSiscomex()
    ConsultandoSiscomex : do / consultarNcm()
    ConsultandoSiscomex : exit / registrarLatencia()
    LevantandoDocumentos : entry / carregarBaseNormativa()
    LevantandoDocumentos : do / buscarRequisitos()
```

---

## 📄 Artefato 4 — Visão Arquitetural C4 Nível 2: Contêineres (Mermaid)

**Objetivo:** representar a topologia e as unidades de implantação do sistema.

**Requisitos técnicos e como foram atendidos**

- ✅ Visão de Contêineres (C4 Nível 2).
- ✅ Ferramenta avançada: Mermaid (`C4Container`).
- ✅ Tecnologias explícitas em cada contêiner (React 18, Node.js 20/NestJS, Python/FastAPI, RabbitMQ 3.13, PostgreSQL 16) e protocolos nos conectores (HTTPS/JSON, gRPC, AMQP, SQL/TCP).

```mermaid
C4Container
    title Diagrama de Contêineres - Assistente de IA para Exportação

    Person(usuario, "Usuário de Exportação", "Exportador ou analista de comércio exterior")
    Person(curador, "Curador Regulatório", "Mantém e valida a base normativa")

    System_Boundary(sistema, "Assistente de IA para Exportação") {
        Container(spa, "Aplicação Web", "React 18, TypeScript", "Chat, consulta de NCM e relatórios de exportação")
        Container(api, "API Gateway (BFF)", "Node.js 20, NestJS", "Orquestra as requisições e autentica usuários")
        Container(cls, "Serviço de Classificação", "Node.js 20", "Consulta e valida NCM (RF05)")
        Container(doc, "Serviço de Análise Documental", "Node.js 20", "Documentos exigidos por país (RF07)")
        Container(frt, "Serviço de Frete", "Node.js 20", "Cotação de frete (RF08)")
        Container(ia, "Motor de IA", "Python 3.12, FastAPI, RAG", "Respostas em linguagem natural sobre normas")
        Container(wrk, "Worker de Atualização Normativa", "Node.js 20", "Coleta e processa mudanças de normas (RF09)")
        ContainerQueue(mq, "Barramento de Eventos", "RabbitMQ 3.13", "Eventos assíncronos de domínio")
        ContainerDb(db, "Base Normativa", "PostgreSQL 16 + pgvector", "Normas, requisitos por NCM/país e embeddings")
    }

    System_Ext(sis, "Siscomex", "Sistema governamental de comércio exterior")
    System_Ext(agentes, "Sites de Agentes de Carga", "Tarifas e prazos de frete")
    System_Ext(mercado, "Fontes de Mercado do Destino", "Normas e exigências dos países importadores")

    Rel(usuario, spa, "Usa", "HTTPS")
    Rel(curador, spa, "Revisa e publica normas", "HTTPS")
    Rel(spa, api, "Chama a API", "HTTPS/JSON (REST)")
    Rel(api, cls, "classificarNcm", "gRPC")
    Rel(api, doc, "levantarDocumentos", "gRPC")
    Rel(api, frt, "cotarFrete", "gRPC")
    Rel(api, mq, "Publica eventos", "AMQP")
    Rel(cls, sis, "Consulta NCM", "HTTPS/JSON (REST)")
    Rel(frt, agentes, "Obtém tarifas", "HTTPS/JSON (REST)")
    Rel(doc, db, "Lê requisitos", "SQL/TCP")
    Rel(doc, ia, "Consulta RAG", "HTTP/JSON")
    Rel(ia, db, "Busca vetorial", "SQL/TCP")
    Rel(mq, wrk, "Entrega eventos", "AMQP")
    Rel(wrk, mercado, "Coleta normas", "HTTPS")
    Rel(wrk, db, "Atualiza normas", "SQL/TCP")
```

---

## 📄 Artefato 5 — Registro de Decisão Arquitetural

**Objetivo:** documentar e justificar a escolha de uma decisão de infraestrutura ou padrão da funcionalidade.

**Estrutura obrigatória (formato Nygard):** Título (`ADR 0001 — [Nome da Decisão]`), Status (Aprovado / Proposto), Contexto (problema técnico ou de negócio), Decisão (tecnologia, protocolo ou padrão adotado e por quê) e Consequências (vantagens e débitos/riscos aceitos).

### ADR 0001 — Adoção de mensageria assíncrona com RabbitMQ para eventos e atualização da base normativa

**Status:** Proposto

**Contexto:**
A análise regulatória depende de fontes externas lentas e instáveis (Siscomex, agentes de carga, fontes de mercado dos países de destino). Além disso, a atualização da base normativa (RF09) é um processo demorado, com revisão humana do curador regulatório, e não pode bloquear a resposta ao usuário. Chamadas síncronas encadeadas aumentariam a latência, acoplariam os serviços e fariam falhas externas derrubarem a análise inteira. Também é necessário avisar o curador quando uma NCM ou norma não for encontrada e reabrir análises já concluídas quando uma norma mudar.

**Decisão:**
Adotar o **RabbitMQ 3.13** (protocolo **AMQP**) como barramento de eventos entre os contêineres. As chamadas que precisam de resposta imediata (classificar NCM, levantar documentos, cotar frete) continuam síncronas via **gRPC**. Os eventos de domínio (`AnaliseConcluida`, `NcmNaoEncontrada`, `NormaAtualizada`) são publicados em *exchanges* do tipo *topic*, com filas duráveis e *dead-letter queue* para mensagens com falha. O RabbitMQ foi escolhido em vez de Kafka por oferecer roteamento flexível, operação simples e volume de mensagens compatível com o porte do sistema, sem exigir *replay* de longo prazo.

**Consequências:**

*Vantagens*
- Desacoplamento entre serviços e o processo de atualização normativa.
- Resiliência: falhas temporárias de fontes externas são tratadas com reentrega e *retry*.
- Resposta ao usuário não fica bloqueada por tarefas demoradas ou por revisão humana.
- Facilidade para adicionar novos consumidores (ex.: auditoria) sem alterar o produtor.

*Débitos e riscos aceitos*
- Mais um componente de infraestrutura para operar, monitorar e proteger.
- Consistência eventual: o relatório pode refletir uma norma ainda não atualizada.
- Necessidade de consumidores idempotentes, pois a entrega é *at-least-once*.
- Maior complexidade de depuração e rastreamento (exige *correlation id* nas mensagens).
