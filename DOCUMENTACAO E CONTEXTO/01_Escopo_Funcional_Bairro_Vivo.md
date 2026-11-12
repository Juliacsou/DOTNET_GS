# BAIRRO VIVO | Escopo funcional consolidado

**Versão:** 1.1 — 10/11/2026  
**Projeto:** FIAP Global Solution 2026/2 — CitizenTech  
**Estado:** Especificação de alinhamento para desenvolvimento. Não é um esquema de banco nem uma API implementada.

## 1. Visão do produto

**Bairro Vivo** é uma plataforma colaborativa de monitoramento de problemas urbanos. Qualquer visitante pode explorar o mapa e acompanhar relatos. Para registrar ocorrências, adicionar informações, publicar fotos, votar sobre a situação ou solicitar uma nova orientação de IA, a pessoa precisa fazer login.

A comunidade — e não um gestor — informa a evolução dos problemas. Duas confirmações válidas, de contas diferentes, permitem encerrar uma ocorrência; duas confirmações válidas de persistência permitem reabri-la. A IA ajuda o cidadão a descobrir caminhos de atendimento em órgãos e serviços públicos, apresentando fontes oficiais quando disponíveis. O projeto **não promete abrir chamados oficiais nem representar a prefeitura**.

### Princípios que já foram decididos

1. Sem tabela ou vínculo obrigatório de bairro. Ocorrências são identificadas por **latitude e longitude**. Município/UF são referências adicionais para orientar a IA e produzir relatórios.
2. **Leitura pública:** mapa, detalhes, fotos, atualizações e histórico visíveis sem login.
3. **Escrita autenticada:** qualquer contribuição exige conta e JWT emitido pelo Java.
4. **Anonimato público opcional:** a publicação pode exibir “Cidadão anônimo”; internamente mantém vínculo com a conta. A ocorrência anônima aparece em **Minhas ocorrências** do autor.
5. Dois estados públicos: **ABERTA** e **RESOLVIDA**. Sem “em análise” ou aprovação de gestor.
6. Duas confirmações de **usuários diferentes** na fase atual fecham/reabrem automaticamente o relato. A mesma conta não fornece os dois votos.
7. **ADMIN:** somente relatórios/indicadores; não altera status nem atua como moderador obrigatório.
8. Java: contas, JWT e TODA a IA de orientação cidadã com Spring AI e RAG; consulta os dados da ocorrência na API .NET. .NET: ocorrências, mapa, fotos, participação comunitaria, status automatico e relatórios; SEM funcionalidades de IA. Oracle: persistencia relacional. MongoDB: entregavel acadêmico de representação documental.
9. Fotos em **armazenamento de objetos/imagens** (provedor a configurar); Oracle armazena referências, vínculos e metadados; não pressupor armazenamento binário em BLOB.

## 2. Atores e permissões

| Ação | Visitante | CIDADÃO autenticado | ADMIN autenticado |
|---|:---:|:---:|:---:|
| Ver mapa, listagem, detalhes, imagens, comentários e histórico | Sim | Sim | Sim |
| Ver orientações de IA disponibilizadas publicamente | Sim | Sim | Sim |
| Criar ocorrência, inclusive publicamente anônima | Não | Sim | Não, no perfil ADMIN do MVP |
| Editar/solicitar exclusão da própria ocorrência | Não | Sim, nas regras de autoria | Não, no perfil ADMIN do MVP |
| Comentar e anexar atualização com foto | Não | Sim | Não, no perfil ADMIN do MVP |
| Indicar “continua” ou “resolvido” | Não | Sim | Não, no perfil ADMIN do MVP |
| Solicitar nova orientação com IA | Não | Sim | Não, no perfil ADMIN do MVP |
| Ver “Minhas ocorrências” e perfil | Não | Sim | Não, exceto perfil pessoal de admin |
| Consultar relatórios administrativos | Não | Não | Sim, somente leitura |
| Alterar manualmente status de uma ocorrência | Não | Não | **Não** |

**Nota de acesso:** no MVP, `ADMIN` e `CIDADAO` são papéis operacionais distintos. O ADMIN consulta somente dados públicos e relatórios; para participar como cidadão, utiliza uma conta com papel CIDADAO. Nunca autorizar mudanças manuais de status com base no papel de admin.

## 3. Funcionalidades planejadas para o MVP

Prioridades: **P0** = imprescindível para apresentação integrada; **P1** = planejado no produto, implementar após P0 caso tempo permita. Os identificadores abaixo são estáveis para frontend, APIs, backlog e testes.

### A. Acesso, conta e preferências — Java

| ID | Prioridade | Funcionalidade | Regra essencial |
|---|---|---|---|
| BV-01 | P0 | Cadastro por nome, email e senha | Email único; validação e senha armazenada como hash |
| BV-02 | P0 | Login/logout real | Java emite JWT; logout limpa a sessão do aplicativo |
| BV-03 | P0 | Visualização pública sem login | Nenhum token exigido em GETs públicos |
| BV-04 | P0 | Meu perfil | Visualizar e editar apenas a própria conta |
| BV-05 | P1 | Alteração e recuperação de senha | Alteração autenticada; recuperação depende de canal de email seguro |
| BV-06 | P1 | Preferências | Dados de interface; não presumir bairro de residência |
| BV-07 | P1 | Locais favoritos | Coordenadas e rótulos como “Trabalho”, opcionais e privados |

### B. Mapa e localização — Frontend + .NET

| ID | Prioridade | Funcionalidade | Regra essencial |
|---|---|---|---|
| BV-08 | P0 | Mapa interativo com marcadores | Coordenadas fornecidas pela API; sem geolocalização obrigatória |
| BV-09 | P0 | Solicitar GPS quando necessário | Permissão em tempo de uso; oferecer seleção manual como alternativa |
| BV-10 | P0 | Explorar livremente o mapa | Sem restrições por bairro ou local de residência |
| BV-11 | P0 | Filtrar por categoria/status | Status apenas ABERTA/RESOLVIDA |
| BV-12 | P0 | Consultar detalhes ao tocar no marcador | Foto, descrição, atualizações, status e data |
| BV-13 | P1 | Ocorrências em um raio selecionado | Geolocalização fornecida ao backend; distâncias no servidor |

### C. Cadastro e gestão da própria ocorrência — .NET

| ID | Prioridade | Funcionalidade | Regra essencial |
|---|---|---|---|
| BV-14 | P0 | Registrar ocorrência | Título, descrição, categoria e latitude/longitude; fotografia opcional |
| BV-15 | P0 | Publicar anonimamente | `anonimoPublico` oculta autor na leitura pública, sem perder autoria interna |
| BV-16 | P0 | Anexar fotografias | Múltiplas fotos por ocorrência, com limites e validação de uploads |
| BV-17 | P0 | Minhas ocorrências | Consulta pelo `sub` do JWT, inclusive registros anônimos |
| BV-18 | P0 | Editar própria ocorrência | Política sugerida: antes da primeira interação comunitária; após isso, complementar via atualização |
| BV-19 | P0 | Excluir/retirar própria ocorrência | Exclusão lógica, com restrições para preservar histórico comunitário; validar regra antes do SQL |
| BV-20 | P0 | Categorias de problemas | Ex.: buracos, iluminação, árvores, lixo, vazamentos, acessibilidade |

### D. Atualizações comunitárias, fotos e votação — .NET

| ID | Prioridade | Funcionalidade | Regra essencial |
|---|---|---|---|
| BV-21 | P0 | Adicionar comentário | Com foto opcional; autoria publicamente anônima opcional |
| BV-22 | P0 | Informar “problema continua” | Pode conter texto e foto; enquanto ABERTA documenta persistência |
| BV-23 | P0 | Informar “problema resolvido” | Pode conter texto e foto; enquanto ABERTA conta voto de resolução |
| BV-24 | P0 | Fechamento comunitário automático | 2 votos válidos de resolução de contas distintas no ciclo atual |
| BV-25 | P0 | Reabertura comunitária automática | 2 votos válidos de persistência de contas distintas no novo ciclo |
| BV-26 | P0 | Histórico e contadores | Exibe comentários, evidências e transições de status, com datas |
| BV-27 | P0 | Impedir voto duplicado | No máximo um voto válido por conta, por ocorrência e ciclo de decisão |

### E. Inteligência artificial - somente Java / Spring AI

| ID | Prioridade | Funcionalidade | Regra essencial |
|---|---|---|---|
| BV-28 | P0 | Solicitar orientação cidadã (Java/Spring AI) | Usuário autenticado seleciona ocorrência existente; Java consulta seus dados na API .NET e recupera conhecimento oficial |
| BV-29 | P0 | Consultar orientação existente | Consulta pública apenas de orientações publicadas sem dados pessoais; solicitação nova requer login |
| BV-30 | P0 | Indicar órgão/serviço competente | Usar município e natureza do problema; informar incerteza se houver atribuição ambígua |
| BV-31 | P0 | Mostrar canais e horários verificados | Telefone, site, endereço e horário apenas com fonte confiável |
| BV-32 | P0 | Mostrar passo a passo e fontes | RAG básico com recuperação de fontes oficiais, data de verificação e referências consultáveis |
| BV-33 | P0 | Mostrar cuidados de segurança pertinentes | Não inventar instruções, nem substituir atendimento emergencial |

**Fluxo da IA:** frontend solicita orientação ao Java passando somente `ocorrenciaId` -> Java obtem categoria, descrição e município da API .NET -> recupera serviços, canais e fontes oficiais em base sob responsabilidade Java -> Spring AI gera resposta fundamentada -> Java devolve estrutura com fontes e disponibilidade. Não existe recurso de melhorar descrição com IA nesta versão. A IA não altera ocorrências, votos ou status e não abre chamado oficial.

**Limite inicial:** fontes verificadas para o município de São Paulo; se o município não puder ser confirmado, ou não houver fontes pertinentes, retornar orientação indisponivel ou parcial, nunca inventar telefone/horário.

### F. Administração informativa — .NET

| ID | Prioridade | Funcionalidade | Regra essencial |
|---|---|---|---|
| BV-34 | P0 | Painel com contadores | Abertas, resolvidas, registros por categoria e período |
| BV-35 | P0 | Relatório com filtros | Município, categoria e período; somente perfil ADMIN |
| BV-36 | P1 | Exportação de relatório | CSV; dados agregados, sem divulgar emails/nomes desnecessariamente |

### G. Apoios de produto e entrega

| ID | Prioridade | Funcionalidade | Regra essencial |
|---|---|---|---|
| BV-37 | P0 | Tela “Sobre o App” | Mostra hash do commit de referência, exigido por Mobile |
| BV-38 | P0 | Feedback e tratamento de erros | Loading, retry, erro de rede, 401, 403, 409, 429 e validação |
| BV-39 | P0 | API real, persistência e CRUD | Dados via Java/.NET; não apenas dados locais |
| BV-40 | P0 | Build e distribuição do app | Publicar no Firebase App Distribution conforme enunciado |

### Fora do MVP

Notificações push; mapa de calor; pesquisa por endereço via geocodificação de terceiros; detecção automática de duplicidades próximas; integração oficial com prefeituras; moderação avançada; roteamento para serviços públicos; acompanhamento de protocolo oficial; expansão da base de IA para todos os municípios brasileiros.

## 4. Ciclo de vida e votação comunitária

- Uma ocorrência nasce **ABERTA**.
- Tipos de atualização: `COMENTARIO`, `PERSISTE`, `RESOLVIDO`. Texto e imagens opcionais, mas pelo menos um conteúdo textual ou imagem é necessário no comentário comum.
- Em ABERTA, uma atualização `RESOLVIDO` pode registrar **um voto**; ao atingir **dois usuários autenticados distintos** no ciclo atual, a API altera para RESOLVIDA.
- Em RESOLVIDA, uma atualização `PERSISTE` pode registrar **um voto**; ao atingir dois usuários distintos no novo ciclo, a API altera para ABERTA.
- `PERSISTE` em ABERTA, ou `RESOLVIDO` em RESOLVIDA, pode acrescentar evidência, sem mudar o status.
- Um usuário não pode contar duas vezes no mesmo ciclo. Repetir clique/requisição não gera voto extra.
- Cada mudança inicia um **novo ciclo de votação**, impedindo reaproveitamento de votos antigos.
- A transição e o registro do histórico devem ocorrer de forma atômica no backend/Oracle.
- O autor da ocorrência também é uma conta autenticada e pode participar, mas nunca sozinho fornece as duas confirmações.
- A participação anônima publicamente continua vinculada internamente ao usuário autenticado.
- A regra de duas contas diferentes **não garante duas pessoas físicas distintas**; trata-se de proteção básica contra manipulação.

**Proposta a validar na etapa do banco:** edição do relato original apenas antes da primeira atualização/voto; exclusão lógica para registros com histórico; eventual remoção de conteúdo abusivo exigiria governança adicional, não incluída no escopo do ADMIN relatórios.

## 5. Dados conceituais (não são DDL nem tipos definitivos)

**Identidade Java:** Usuário → Preferências (1:1) e Locais favoritos (1:N). A identidade canônica via JWT deve ter o mesmo formato reconhecido pela API .NET e pelo Oracle.  
**Domínio .NET:** Categoria → Ocorrência → Atualizações → Mídias; Ocorrência → Votos por ciclo → Histórico de status.  
**IA/serviços (dominio Java):** Município -> Órgãos -> Serviços -> Canais -> Fontes oficiais; Orientação gerada -> referência para ocorrência .NET e fontes usadas. Os dados de ocorrências permanecem sob responsabilidade .NET; Java pode consumir sua API de leitura, mas não escreve nas tabelas de ocorrências.

O cadastro de ocorrências **não tem `ID_BAIRRO`**, e a pessoa **não precisa informar endereço de residência**. Latitude/longitude são obrigatórias no ponto da ocorrência. Município/UF podem ser obtidos por preenchimento assistido ou serviço de geocodificação quando disponível; para orientar a IA, a identificação do município deve ser confiável.

## 6. Medidas de resultado e limitações

Indicadores: quantidade de problemas publicados, quantidade e percentual de resolvidas, tempo até resolução comunitária, categorias frequentes, regiões com concentração, número de atualizações/participações, consultas à orientação de IA.

**Limitações transparentes:** confirmação comunitária não equivale à confirmação oficial; telefones/horários públicos podem mudar; a solução não abre protocolos automaticamente; municípios fora da base de fontes verificadas podem receber orientação limitada ou resposta de fonte indisponível.

## 7. Próxima etapa deliberadamente não executada

Já existe um script preliminar Oracle v1.0, criado antes desta redistribuicao de responsabilidades. Ele precisa ser revisado para alocar as tabelas de orientação e conhecimento oficial ao dominio Java e remover expectativas de IA no .NET. Este documento define a arquitetura desejada, não afirma que o script antigo já esteja atualizado nem validado em Oracle.
