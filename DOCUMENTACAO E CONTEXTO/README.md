# Pacote de alinhamento — Bairro Vivo v1.1 (GS 2026/2)

**Versão:** 1.1 — 10/11/2026.

## Arquivos

1. **01_Escopo_Funcional_Bairro_Vivo.md** — visão do produto, requisitos, permissões, estados, votação e prioridades.
2. **02_Documentacao_Frontend_React_Native.md** — telas, navegação, mapas/GPS, fotos, integração com duas APIs, autenticação e critérios de aceite.
3. **03_Especificacao_Backend_Java_Spring_Boot.md** — responsabilidades Java, entidades conceituais, endpoints propostos, JWT compatível com .NET, Spring AI, mensageria e testes.

## Situação atual

- Oracle é o banco principal; existe uma primeira versão do SQL, ainda não validada em execução Oracle e pendente de revisão quanto à IA.
- As rotas e contratos de API são **sugestões de desenvolvimento**, não endpoints existentes.
- Java controla cadastro, login, JWT, preferências, favoritos e **toda a IA de orientação cidadã com Spring AI/RAG**.
- .NET controla ocorrências, mapa, fotos, votos comunitários, status e relatórios, **sem IA**.
- Visitantes **apenas consultam**; contribuições exigem login.
- Publicações podem aparecer como anônimas publicamente e continuam vinculadas internamente ao autor.
- Somente dois status: ABERTA e RESOLVIDA, decididos com duas confirmações válidas de contas distintas em cada ciclo.
- Perfil ADMIN vê relatórios, sem gerenciar ocorrências.

**Atenção:** a proposta PDF inicial era anterior a estas definições e contém conceitos revogados (bairros, gestor de status e IA de triagem). Para desenvolvimento, prevalecem estes três documentos.

## Compatibilidade com o SQL preliminar

Existe um SQL Oracle v1 preliminar, criado com a antiga distribuição da IA. Este pacote **não altera esse SQL**. Antes de adotá-lo, revisar propriedade das tabelas de órgãos/serviços/fontes/orientações (Java), contratos de consulta Java -> .NET e IDs/relacionamentos. Nenhum endpoint ou tipo físico novo foi validado como implementado.

## Fontes acadêmicas

Material fornecido pelo grupo: **2TDS Fevereiro - Global Solution 2026-2.pdf**, em especial as seções Java Advanced, Mobile Application Development, Advanced Business Development with .NET e Mastering Relational and Non-Relational Database.

## O que mudou na versão 1.1

- A **única IA** do aplicativo é **Orientação Cidadã** em Java/Spring AI, com RAG e fontes oficiais.
- **Removida** a proposta do botão de melhorar descrição da ocorrência.
- .NET continua responsável por ocorrências, fotos, votos, status e relatórios, sem modelos de IA.
- Java consulta a ocorrência via API .NET de leitura antes de gerar a orientação; o frontend chama Java para a resposta.
- Documentos de escopo, frontend e Java foram alinhados. O script Oracle v1 ainda requer revisão posterior.
