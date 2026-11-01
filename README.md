# AI Operations: Triagem Inteligente de Chamados com Low-Code e IA

Automação inteligente para o recebimento, classificação e resposta de chamados de alunos em uma universidade, construída com **Make**, **Google Forms/Sheets**, **Gemini AI** e **Trello**, com supervisão humana (*Human-in-the-Loop*) nos casos de urgência alta.

> Trabalho da disciplina **Desenvolvimento de Aplicações e Automações com Low Code**
> Curso de Inteligência Artificial e Automação Digital, 3º semestre, UniFECAF
> Autora: **Priscylla Pfleger**

---

## Contexto e problema

Em uma universidade em crescimento, o recebimento de chamados envolve diferentes sistemas, tarefas repetitivas e decisões manuais. Os chamados chegam por formulários e precisam ser analisados, registrados, classificados, encaminhados e acompanhados até a conclusão. Grande parte desse processo depende de pessoas transferindo informações entre ferramentas e atualizando dados manualmente, o que:

- aumenta o tempo de execução;
- gera retrabalho;
- dificulta a rastreabilidade;
- limita a capacidade de crescimento da operação.

## Solução proposta

Uma **Automação Inteligente** capaz de:

1. receber as demandas dos alunos;
2. usar IA para interpretar o texto, classificar por **área** e por **urgência** e gerar uma **resposta prévia**;
3. responder pelo fluxo normal ou direcionar para atendimento humano quando necessário (urgência **Alta/Crítica**).

Nesta primeira versão, a automação separa as demandas de urgência **Alta** das demais. As de urgência Alta passam por análise, supervisão e aprovação humana; as de urgência Baixa ou Média seguem pelo fluxo normal com resposta registrada automaticamente.

## Fluxograma do processo

| # | Etapa | Descrição |
|---|-------|-----------|
| 1 | **Entrada (gatilho)** | Formulário preenchido pelo aluno (Google Forms) |
| 2 | **Processamento por IA** | O texto é enviado ao Gemini, que devolve categoria, urgência e resposta prévia |
| 3 | **Decisão e HITL** | Um cartão é criado no Trello na coluna "Entrada para verificação" com o texto original, a classificação e o rascunho; o responsável ajusta e move para "Aprovado" |
| 4 | **Ação** | O movimento do cartão dispara o registro da resposta final e o envio ao aluno |
| 5 | **Registro (persistência)** | Categoria, urgência e texto final são gravados no Google Sheets para relatórios e auditoria |

## Stack utilizada

| Ferramenta | Função |
|------------|--------|
| **Google Forms** | Captura do chamado pelo aluno |
| **Google Sheets** | Controle e histórico de todas as demandas |
| **Google Apps Script** | Envia os dados da planilha ao Webhook do Make |
| **Make** | Orquestração do fluxo |
| **Gemini AI** (modelo Flash) | Classificação por área, classificação por urgência e geração da resposta prévia |
| **Trello** | Fila de verificação e aprovação humana (HITL) |

## Arquitetura da automação

### Cenário 1: "Triagem chamados"

```
Webhook → Gemini (Área) → Gemini (Urgência) → Sheets (atualiza linha)
        → Gemini (Resposta prévia) → Condicional de urgência
                                        ├─ Alta  → Trello (cria card para aprovação)
                                        └─ Demais → Sheets (registra resposta)
```

1. **Webhook:** recebe o payload enviado pelo Apps Script.
2. **Triagem 1, Classificação de Área:** Gemini classifica em *Pedagógica*, *Financeira* ou *Administrativa/Outras*.
3. **Triagem 2, Classificação de Urgência:** Gemini classifica em *Alta*, *Média* ou *Baixa*.
4. **Update a Row (Sheets):** grava área e urgência na planilha.
5. **Resposta prévia:** Gemini gera um rascunho de resposta.
6. **Condicional de urgência:**
   - **Alta:** cria um card no Trello para aprovação humana.
   - **Baixa/Média:** registra a resposta na planilha, que pode ser enviada ao aluno por mensagem de texto ou e-mail.

### Cenário 2: "Triagem chamados - parte 2" (após aprovação)

```
Webhook (Trello) → Get a Card → Match pattern (identifica a linha) → Sheets (registra resposta)
```

Disparado quando um card é movido para a lista **APROVADOS**. O fluxo recupera o card, identifica a linha da planilha e registra a resposta final.

## Configuração passo a passo

### 1. Webhook no Make

Crie um módulo **Custom Webhook** no cenário e copie a URL gerada. O webhook detecta os campos: `row`, `DataHora`, `E-mail`, `Nome`, `Matrícula` e `Demanda`.

### 2. Apps Script na planilha

Na planilha vinculada ao Google Forms, acesse **Extensões → Apps Script** e adicione o código abaixo (substitua a URL pelo seu webhook):

```javascript
function enviarParaOMake(e) {
  var urlWebhook = "https://hook.us2.make.com/SEU_TOKEN_AQUI";
  var linhaEditada = e.range.getRow();
  var aba = e.range.getSheet();
  var valores = aba.getRange(linhaEditada, 1, 1, aba.getLastColumn()).getValues()[0];

  var payload = {
    "row": linhaEditada,
    "DataHora": valores[0] ? valores[0].toString() : "",
    "E-mail":   valores[1] ? valores[1].toString() : "",
    "Nome":     valores[2] ? valores[2].toString() : "",
    "Matrícula": valores[3] ? valores[3].toString() : "",
    "Demanda":  valores[4] ? valores[4].toString() : ""
  };

  var options = {
    "method": "post",
    "contentType": "application/json",
    "payload": JSON.stringify(payload)
  };

  UrlFetchApp.fetch(urlWebhook, options);
}
```

### 3. Acionador (trigger) do Apps Script

Em **Acionadores**, crie um gatilho com:

- **Função:** `enviarParaOMake`
- **Origem do evento:** Da planilha
- **Tipo de evento:** Ao enviar o formulário

### 4. Regra de automação no Trello

No quadro "Chamados", crie duas listas: **Entrada para verificação** e **APROVADOS**. Em **Automação → Regras**, configure:

- **Gatilho:** quando um card for movido para a lista `APROVADOS`
- **Ação:** *post to URL* com a URL do Webhook do Make e o payload:

```json
{"card_id": "{cardid}", "card_name": "{cardname}", "card_desc": "{carddesc}"}
```

### 5. Conteúdo do card no Trello

O card contém: nome do aluno, e-mail, área classificada (para encaminhar ao setor correspondente), demanda do aluno e o número da linha da planilha (para registrar o retorno depois). A descrição traz a resposta prévia gerada pela IA.

## Prompts utilizados

### Etapa 1: Classificação por área

**Instrução do sistema:**

> Você é um assistente educacional especialista em triagem de chamados e atendimentos. Sua única função é analisar a demanda do aluno, realizar a classificação categórica, avaliar a urgência e gerar um rascunho resposta em texto simples, motivacional e direto no tom de voz.

**Mensagem (Role: User, Type: Text):**

```
Você é um triador acadêmico. Analise a demanda do aluno abaixo e classifique-a estritamente em uma das três áreas principais.

Categorias permitidas:
- Pedagógica
- Financeira
- Administrativa/Outras

Demanda do Aluno: "{{46.Demanda}}"

Retorne apenas a área da demanda classificada.
```

### Etapa 2: Classificação por urgência

**Instrução do sistema:**

> Você é um analista de risco e priorização de atendimento especialista em triagem de chamados e atendimentos. Sua única função é analisar a demanda do aluno, realizar a classificação categórica, avaliar a urgência e gerar um rascunho resposta em texto simples, motivacional e direto no tom de voz.

**Mensagem (Role: User, Type: Text):**

```
Você é um analista de risco e priorização de atendimento. Com base na mensagem do aluno e na área já identificada ({{26.result}}), defina o nível de urgência.

Critérios:
- Alta: Prazos fatais (hoje/amanhã), impedimentos graves de acesso à plataforma e a exames, cancelamentos.
- Média: Solicitações administrativas padrão com prazo regular de atendimento.
- Baixa: Dúvidas gerais, informações simples sem dependência de prazo.

Demanda do Aluno: "{{46.Demanda}}"

Retorne apenas a urgência definida para a demanda classificada.
```

### Etapa 3: Resposta prévia

**Mensagem (Role: User, Type: Text):**

```
Atue como assistente administrativo para solucionar demandas de chamados estudantis.
Gere uma resposta sucinta, simples, mas completa, para responder/atender ao chamado recebido ({{46.Demanda}}).
```

> Os números entre chaves (`{{46.Demanda}}`, `{{26.result}}`) são referências a módulos do cenário no Make e podem mudar no seu ambiente.

## Human-in-the-Loop (HITL)

Demandas de urgência **Alta** não são respondidas automaticamente. O fluxo é:

1. Um card é criado na lista **Entrada para verificação** do Trello.
2. O atendente do setor correspondente revisa e corrige a resposta prévia da IA.
3. Ao mover o card para **APROVADOS**, o segundo cenário é disparado.
4. A resposta final é registrada na planilha e enviada ao aluno.

Isso mantém a IA como apoio à decisão, com validação humana nos casos de maior impacto.

## Registro e auditoria

A planilha **Histórico de Chamados** (Google Sheets) concentra, para cada chamado:

- data/hora, e-mail, nome completo e matrícula;
- descrição da demanda;
- área e urgência classificadas;
- retorno (resposta final) registrado.

Isso garante rastreabilidade e base para relatórios.

## Segurança e privacidade

- **Minimização de dados:** o formulário coleta apenas o necessário para identificação, tratamento e retorno (sem dados sensíveis como CPF).
- **Credenciais no Make:** chaves de API e tokens ficam armazenados de forma criptografada no gerenciador interno de credenciais (*Keychain*), isolados da lógica visual dos cenários e fora dos logs de execução.
- **Webhooks:** o Webhook customizado gera URL única com token aleatório e permite exigir autenticação por chave de API no cabeçalho; também é possível restringir os IPs de origem autorizados.
- **Menor privilégio:** tokens de acesso devem ter apenas as permissões estritamente necessárias.
- **Chaves do Google AI Studio / Cloud:** devem ser restritas à API generativa; o envio recomendado é pelo cabeçalho HTTP `x-goog-api-key` ou pelos SDKs oficiais. O Google Cloud Console e o AI Studio oferecem rastreamento de uso por credencial.
- **Dados no Gemini:** prompts e respostas em ambientes corporativos ou via Cloud não são usados para treinar os modelos públicos do Google, e os dados são criptografados em trânsito e em repouso.

> **Nunca versione a URL real do webhook nem chaves de API.** Use placeholders no repositório.

## Próximos passos

- Ampliar a automação com novas ações conforme os colaboradores se adaptarem ao processo.
- Implementar as **subclasses** por área (Pedagógico, Financeiro e Administrativo) previstas no fluxograma.
- Automatizar o envio final da resposta ao aluno por e-mail ou mensagem de texto.
- Gerar relatórios a partir do histórico da planilha.

## Conclusão

A automação é pequena, mas funcional, e demonstra como IA e Low-Code podem apoiar processos repetitivos e de grande volume, reduzindo o tempo de resposta e o retrabalho e melhorando o desempenho das atividades de qualquer área.

---
