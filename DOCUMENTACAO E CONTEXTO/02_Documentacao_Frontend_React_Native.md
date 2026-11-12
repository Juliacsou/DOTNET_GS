# BAIRRO VIVO | Especificação do frontend (React Native)

**Versão:** 1.1 — 10/11/2026  
**Público:** Desenvolvedor(a) Mobile/Frontend e designer.  
**Objetivo:** Construir telas, navegação, estados, modelos de dados e integração com Java/.NET de maneira consistente **antes de existirem endpoints definitivos**. O SQL Oracle v1 preliminar ainda não reflete integralmente a nova divisão da IA.

> Fonte de requisitos acadêmicos: “2TDS Fevereiro - Global Solution 2026-2.pdf”, seção Mobile Application Development. Exige React Native, pelo menos seis telas com navegação, CRUD usando API real, login/cadastro/logout, identidade visual, vídeo, Firebase App Distribution e tela “Sobre o App” com hash do commit.

## 1. O produto em uma frase

Cidadãos visualizam problemas urbanos no mapa, registram ocorrências com localização e fotos, atualizam registros de outras pessoas e decidem coletivamente quando um problema foi resolvido. A IA orienta como buscar o serviço público apropriado com base em informações oficiais verificadas.

**Não existe vínculo obrigatório com bairros**, não existe gestor para aprovar status, não existe criação de protocolos oficiais e visitantes não realizam ações de escrita.

## 2. Arquitetura de integração

- **Frontend:** React Native com Expo e Expo Router ou React Navigation.
- **API Java:** autenticação e conta, JWT, perfil, preferências, favoritos e TODA a IA de orientação cidadã usando Spring AI e RAG (recuperação de fontes oficiais).
- **API .NET:** mapa/ocorrências, categorias, comentários, fotos, votos, status comunitário, histórico, dados geográficos da ocorrência e relatórios. Não executa IA.
- **Oracle:** persistência das informações relacionais dos dois domínios.
- **Imagens:** serviço externo de armazenamento de imagens (Cloudinary é a opção inicial, sujeita à configuração). O frontend envia através do fluxo definido pela API .NET; não incorpora credenciais secretas.
- **Mapa:** dados OpenStreetMap e renderizador como MapLibre para React Native, com provedor de tiles compatível com a política de utilização. **MapLibre React Native pode exigir development build nativa; não assumir Expo Go.** GPS via Expo Location.

O frontend nunca acessa diretamente Oracle, não calcula a mudanca de status e não cria telefones/horários da IA. A consulta da ocorrência para gerar orientação ocorre entre Java e .NET, pelo backend.

### Autenticação Java → .NET

1. Java recebe credenciais e devolve JWT + dados públicos da conta.
2. O app armazena o JWT com armazenamento seguro apropriado ao dispositivo (ex.: Expo SecureStore no mobile).
3. Requisições autenticadas às duas APIs levam `Authorization: Bearer <token>`.
4. A .NET valida o token emitido pelo Java; o frontend **não envia `idUsuario` para declarar autoria**.
5. Sem sessão: permite GET público, mas redireciona para login quando o visitante toca em registrar, comentar, votar, enviar foto ou solicitar nova IA.
6. Expirou JWT: bloqueia ação, explica expiração e encaminha ao login; logout limpa o token local e estados privados.

**Publicação anônima não é acesso sem login.** A tela oferece um checkbox `Publicar anonimamente` a usuários logados. A API recebe a flag, preserva vínculo de autoria e devolve somente “Cidadão anônimo” nas respostas públicas. A ocorrência continua acessível em “Minhas ocorrências”.

## 3. Matriz de ações e permissões

| Funcionalidade | Visitante | CIDADÃO | ADMIN |
|---|:---:|:---:|:---:|
| Mapa, listagem, filtro, detalhe, imagens e histórico | Sim | Sim | Sim |
| Orientação de IA previamente disponibilizada publicamente | Sim | Sim | Sim |
| Registrar/editar sua ocorrência | Login exigido | Sim | Não, perfil ADMIN apenas relatórios |
| Comentar, publicar fotos, indicar “continua” ou “resolvido” | Login exigido | Sim | Não, perfil ADMIN apenas relatórios |
| Solicitar nova orientação com IA | Login exigido | Sim | Não, perfil ADMIN apenas relatórios |
| Perfil, minhas ocorrências, preferências, favoritos | Login exigido | Sim | Somente perfil pessoal do admin |
| Relatórios consolidados | Não | Não | Sim |
| Atualizar status manualmente | Não | Não | Não |

**Orientações compartilhadas publicamente** não podem exibir dados internos do solicitante. Se nenhuma orientação pública estiver disponível, a página mostra a opção de fazer login para solicitar uma nova.

## 4. Navegação e telas planejadas

| Tela | Rota ilustrativa | Visibilidade | Conteúdo/ações |
|---|---|---|---|
| 1. Mapa / Início | `/mapa` | Pública | Marcadores, GPS, filtros, botão para criar |
| 2. Explorar/listagem | `/ocorrencias` | Pública | Lista, paginação, filtros |
| 3. Detalhe da ocorrência | `/ocorrencias/[id]` | Pública | Fotos, situação, local, atualizações, orientações existentes |
| 4. Login | `/entrar` | Pública | Email, senha, entrar |
| 5. Cadastro | `/cadastro` | Pública | Nome, email, senha, confirmação |
| 6. Nova ocorrência | `/nova-ocorrencia` | Protegida | Categoria, título, texto, fotos, GPS/pino, anonimato |
| 7. Editar minha ocorrência | `/minhas/[id]/editar` | Autor | Ajustar campos permitidos pela API |
| 8. Adicionar atualização | `/ocorrencias/[id]/atualizar` | Protegida | Tipo, comentário, fotos e anonimato |
| 9. Minhas ocorrências | `/minhas` | Protegida | Histórico de registros do próprio usuário, inclusive anônimos |
| 10. Orientação cidadã | `/ocorrencias/[id]/orientacao` | Leitura pública; solicitar requer login | Órgão, serviço, contato, horário, passos, fontes |
| 11. Perfil | `/perfil` | Protegida | Dados da conta, senha, preferências e logout |
| 12. Locais favoritos | `/favoritos` | Protegida | Lista, inclusão, edição e remoção |
| 13. Relatórios | `/admin/relatorios` | ADMIN | Indicadores e filtros, somente leitura |
| 14. Sobre o app | `/sobre` | Pública | Versão, projeto, hash do commit |

Um aplicativo com navegação por abas pode usar `Mapa`, `Explorar`, `Minhas` e `Perfil`. O botão flutuante de registro abre login caso não haja sessão. A aba de admin só aparece para papel autorizado. Não é necessário implementar 14 telas totalmente independentes se uma mesma tela atender os fluxos; **a exigência acadêmica é pelo menos seis telas distintas reais**.

## 5. Fluxos detalhados

### 5.1 Primeiro acesso e visitante

- Carregar o mapa sem login.
- Não solicitar permissão de GPS automaticamente na abertura; oferecer botão **Minha localização**.
- Se GPS for recusado, usar um centro de mapa inicial configurado (para a demo, São Paulo) e permitir arrastar o mapa.
- Carregar marcadores por área visível; quando a câmera se move, consultar novamente com debounce, evitando chamadas em cada pixel de movimento.
- Abrir detalhes com fotos, comentários, número de confirmações e situação.
- Ao tentar ação protegida, mostrar texto curto (“Entre para contribuir com a comunidade”) e preservar a intenção para voltar ao fluxo após autenticar.

### 5.2 Cadastro e login

- Cadastro: validar campos, confirmar senha, mostrar erros específicos.
- Login: indicar carregamento, tratar credencial inválida e sessão expirada.
- Salvar JWT apenas em armazenamento seguro; jamais registrar token em logs ou analytics.
- Logout: excluir token e cache de perfil; redirecionar ao mapa público.
- Se autenticação expirar durante um formulário, preservar rascunho **sem fotos sensíveis indevidamente expostas** e solicitar login novamente.

### 5.3 Criar ocorrência

1. Usuário autenticado toca em **Registrar ocorrência**.
2. Informa categoria, título e descrição livremente. Nesta versão não existe ferramenta de melhoria textual por IA.
3. Define a localização por GPS ou move o pino manualmente; o GPS é opcional, mas **as coordenadas da ocorrência são obrigatórias**.
4. Pode anexar fotos, com prévia e remoção antes de enviar. Fotos são **opcionais**.
5. Marca ou desmarca **Publicar anonimamente** (opcional).
6. Envia para .NET; mostrar loading e evitar envio duplicado; a API garante idempotência.
7. Ao sucesso, abre detalhes com status `ABERTA` e a ocorrência aparece em `Minhas ocorrências`, mesmo se anônima.

**Edição e exclusão:** apenas autoria reconhecida pela API. Proposta de regra de produto: editar campos originais até a primeira interação de terceiros; após isso, publicar uma nova atualização para manter a integridade do histórico. Retirar/excluir via exclusão lógica, nunca simplesmente apagar votos e histórico. Não assumir que um botão disponível significa autorização: backend é o árbitro.

### 5.4 Comentar, atualizar e votar

Na tela de detalhe, o botão **Adicionar atualização** mostra três escolhas:

- **Informação adicional:** comentário e/ou foto; não interfere no status.
- **O problema continua:** comentário e/ou foto; se a ocorrência estiver RESOLVIDA, representa voto de reabertura.
- **O problema foi resolvido:** comentário e/ou foto; se estiver ABERTA, representa voto de resolução.

A pessoa pode escolher **Publicar anonimamente** na atualização, mesmo que a ocorrência original tenha sido pública ou vice-versa.

**Indicador de decisão:** exibir `1 de 2 confirmações para resolver` ou `1 de 2 confirmações para reabrir`, conforme status e dados retornados pela API. Ao atingir 2 contas distintas no ciclo, o backend altera o status. A UI apenas atualiza a tela; **nunca muda status localmente de forma definitiva**.

Quando um usuário já votou neste ciclo, mostrar a ação como concluída ou desabilitada; se ele repetir a requisição, o servidor não poderá contar novamente.

**Regra específica:** enquanto a ocorrência estiver ABERTA, “continua” pode documentar persistência, mas não conta como voto de reabertura. Enquanto RESOLVIDA, “resolvido” pode documentar evidências, mas não cria novo ciclo.

### 5.5 IA de orientação cidadã (Java / Spring AI)

- A tela de orientação pertence a uma ocorrência existente. O frontend recebe o identificador real dela ao abrir o detalhe na API .NET.
- A pessoa sem login pode consultar somente orientações que o Java disponibilizou como públicas; a criação de nova orientação exige token com papel CIDADAO.
- Ao tocar em **Como resolver este problema?**, o frontend envia para Java o `ocorrenciaId` (não envia relato livre editado nem contato imaginado). O Java busca do .NET os dados canônicos: descrição, categoria e município da ocorrência.
- Java recupera fontes oficiais pertinentes para a cidade e gera a resposta via Spring AI/RAG. A UI exibe loading (`Consultando fontes oficiais...`) e depois usa o estado devolvido pelo Java: `DISPONIVEL`, `PARCIAL`, `SEM_FONTE`, `LOCALIDADE_NAO_CONFIRMADA` ou `ERRO` (valores conceituais a validar no contrato).
- Quando confirmados, exibir órgão responsável, tipo de serviço, contatos, horários, passos sugeridos, cuidados de segurança e **fontes clicáveis** com data de verificação. Campos não verificados devem aparecer como `Informacao nao confirmada` ou ser omitidos.
- Em caso de indisponibilidade, mostrar estado vazio/erro e canal de consulta oficial geral se houver fonte confiável; nunca inventar telefone ou horário no frontend.
- A IA não cadastra ocorrência nem altera status; o aplicativo não abre protocolo oficial automaticamente. Em risco imediato, fornecer aviso de precaucao, sem orientação técnica perigosa.
- Uma orientação já gerada só pode ser exibida publicamente se sua resposta estiver marcada pelo Java como publicável, sem PII e com fontes; consultas pessoais não podem vazar a identidade do solicitante.
- Frontend deve manter dois clientes HTTP separados: `api-java` para a orientação e `api-dotnet` para a ocorrência. O fluxo entre APIs ocorre no servidor Java, não no celular.

**Caminho visual:** Detalhe de ocorrência (.NET) -> tela de orientação -> POST para Java -> tela de resultado (Java) -> fontes oficiais externas.

### 5.6 Minhas ocorrências, anonimato e perfil

- A lista `Minhas ocorrências` usa a identidade da sessão (não filtros por nome publicamente exibido).
- Exibir inclusive ocorrências criadas com `anonimoPublico = true`.
- No detalhe, a resposta pública poderá mostrar `Cidadão anônimo`; na visualização pessoal, a interface sabe que o usuário é o autor por um campo explícito de permissões (`souAutor`) ou pelo endpoint de Minhas Ocorrências — nunca por tentativa de adivinhar IDs ocultos.
- Comentários anônimos próprios podem ser destacados no próprio painel privado sem revelar a identidade a outras pessoas.

### 5.7 Relatórios ADMIN

- Rota exclusiva de ADMIN. Indicadores agregados por categoria, status, município e período.
- Sem ações de alteração de status, edição de relato ou remoção de publicação.
- Não expor emails, hashes, usuários participantes nem dados sensíveis nos gráficos.

## 6. Contratos de dados esperados (ilustrativos; não são endpoints definitivos)

A equipe deve trabalhar com **interfaces/tipos estáveis** e um adaptador HTTP. Exemplos de campos a negociar:

```ts
export type StatusOcorrencia = 'ABERTA' | 'RESOLVIDA';
export type TipoAtualizacao = 'COMENTARIO' | 'PERSISTE' | 'RESOLVIDO';

export interface OcorrenciaResumo {
  id: string;
  titulo: string;
  categoria: { id: string; nome: string; icone?: string };
  latitude: number;
  longitude: number;
  status: StatusOcorrencia;
  fotoCapaUrl?: string | null;
  dataCriacao: string; // ISO 8601
  totalAtualizacoes: number;
}

export interface OcorrenciaDetalhe extends OcorrenciaResumo {
  descricao: string;
  autorExibicao: string; // "Cidadão anônimo" ou nome permitido
  anonimoPublico: boolean;
  enderecoReferencia?: string | null;
  municipio?: { codigoIbge?: string; nome: string; uf: string };
  fotos: { id: string; url: string; textoAlternativo?: string }[];
  contagemParaMudarStatus: { atual: number; necessario: 2; alvo: StatusOcorrencia };
  souAutor?: boolean; // somente em resposta privada/autenticada
}

export interface Atualizacao {
  id: string;
  ocorrenciaId: string;
  tipo: TipoAtualizacao;
  comentario?: string | null;
  fotos: { id: string; url: string }[];
  autorExibicao: string;
  anonimoPublico: boolean;
  dataCriacao: string;
  contabilizadaComoVoto: boolean;
}

export interface SessaoAutenticada {
  accessToken: string;
  tokenType: 'Bearer';
  expiresIn: number;
  usuario: { id: string; nome: string; email: string; papeis: string[] };
}

// Resposta da API Java. Campos e status ainda sujeitos a contrato OpenAPI.
export type EstadoOrientacao =
  | 'DISPONIVEL' | 'PARCIAL' | 'SEM_FONTE'
  | 'LOCALIDADE_NAO_CONFIRMADA' | 'ERRO';

export interface FonteOficial {
  titulo: string;
  url: string;
  verificadaEm?: string | null;
}

export interface OrientacaoCidada {
  id: string;
  ocorrenciaId: string;
  estado: EstadoOrientacao;
  municipio?: { nome: string; uf: string } | null;
  orgaoResponsavel?: string | null;
  servico?: string | null;
  canais: { tipo: 'TELEFONE' | 'SITE' | 'PRESENCIAL' | 'OUTRO'; valor: string; horario?: string | null }[];
  passos: string[];
  cuidados: string[];
  fontes: FonteOficial[];
  geradaEm: string;
}
```

**Decisões pendentes até fechar o contrato atualizado e adaptar o Oracle v1:** formato exato dos IDs (UUID/NUMBER), paginação e ordenação, formato dos uploads (multipart ou upload assinado), política de exclusão, relação entre atualizações e votos, apresentação de múltiplos papéis. Não fixar esses detalhes como se já estivessem implementados.

## 7. Requisitos de mapa e GPS

1. Mostrar mapa com informações geográficas do OpenStreetMap e renderizador apropriado para React Native. **OSM é dado cartográfico, não um serviço ilimitado de tiles**; escolher hospedagem compatível e respeitar atribuição e política de uso.
2. Armazenar/usar coordenadas em WGS84: latitude no intervalo `[-90, 90]`, longitude em `[-180, 180]`.
3. Início sem GPS: mapa em posição de demonstração configurável, por exemplo São Paulo; não exigir local de residência.
4. `Minha localização`: só solicitar autorização na interação; mostrar estado de permissão negada e alternativa de seleção manual.
5. Cadastro: pino ajustável; o ponto selecionado é o campo obrigatório. Endereço textual é informativo e opcional.
6. Buscar ocorrências da **área visível (bounding box)**, com filtro de categoria/status; limitar quantidade de marcadores por requisição e considerar agrupamento quando houver muitos pontos.
7. Ao clicar no marcador, abrir resumo com acesso ao detalhe; deixar controles acessíveis via leitor de tela.
8. Se houver busca por texto/geocodificação reversa, tratá-la como melhoria futura dependente de provedor e limites próprios.
9. Evitar divulgar a posição atual do usuário para outros usuários: somente coordenadas **da ocorrência** entram na publicação.
10. Não expor EXIF/GPS original das fotos enviadas; preferir remoção dos metadados no backend de upload.

## 8. Fotos e envio de mídia

- Imagens de ocorrência e imagens de atualização são recursos relacionados, não o arquivo bruto dentro do Oracle.
- O frontend seleciona/captura imagens, gera miniaturas locais para prévia e envia pelo fluxo aprovado com a API .NET.
- **Proposta de limites para MVP:** até 3 fotos por ocorrência ou atualização, até 5 MB por foto, formatos JPEG/PNG/WebP. São valores a confirmar na implementação, não regras já existentes do banco.
- Exibir andamento do envio, possibilidade de remover antes de publicar e erro de upload.
- Não colocar chave secreta do Cloudinary no app; preferir o backend coordenar ou assinar uploads.
- URLs podem mudar ou expirar: usar campos de mídia da API, não persistir URLs públicas como identificador único.

## 9. Estados e tratamento de erros

| Cenário | Comportamento esperado |
|---|---|
| Carregando marcadores | Indicador de carregamento sem travar o mapa |
| Sem ocorrências na região | Estado vazio: “Nenhuma ocorrência nesta área” |
| GPS desativado/negado | Usar localização manual; nunca bloquear exploração |
| `401` em ação | Fazer login novamente / restaurar sessão válida |
| `403` | Informar falta de permissão sem mostrar dados privados |
| `404` | Mostrar “Registro indisponível” e opção de voltar |
| `409` (voto duplicado/edição bloqueada) | Mostrar regra da comunidade e recarregar a ocorrência |
| `429` | Mensagem de limite temporário; permitir tentar novamente mais tarde |
| `5xx` ou offline | Retry, mensagem compreensível e preservação de rascunho |
| Falha de IA/fonte | Exibir indisponibilidade sem gerar orientação inventada |

## 10. Estrutura de código sugerida

```text
src/
  app/                 # rotas / navegação
  components/          # cartões, marcadores, campos, botões
  features/
    auth/
    mapa/
    ocorrencias/
    atualizacoes/
    orientacao-ia/
    perfil/
    favoritos/
    admin-relatorios/
  services/
    api-java.ts
    api-dotnet.ts
    auth-storage.ts
    media-upload.ts
  models/              # interfaces e enums
  hooks/               # localização, sessão, filtros
  theme/               # cores, fontes, estilos
  assets/
```

Dois clientes HTTP com URLs de base distintas. Interceptador de autenticação só insere Bearer em chamadas que precisam de sessão; GETs públicos funcionam sem token. Preferir chamadas reais em homologação e mocks apenas em desenvolvimento de tela.

## 11. Critérios de aceite essenciais

- [ ] Visitante visualiza mapa e detalhes sem login.
- [ ] Visitante tocando em “Registrar/Comentar/Confirmar” é encaminhado ao login.
- [ ] Cadastro, login e logout reais com Java; rotas privadas bloqueadas após logout.
- [ ] Usuário autenticado cria ocorrência com coordenadas selecionadas, sem foto obrigatória.
- [ ] Autor consulta ocorrência anônima na lista **Minhas ocorrências**.
- [ ] Outro usuário consegue adicionar comentário/foto e atualizar a situação.
- [ ] Duas contas distintas resolvem uma ocorrência; outras duas atualizações válidas podem reabri-la.
- [ ] A UI não permite contagem duplicada nem falsifica status localmente.
- [ ] Ao solicitar orientação, o frontend chama Java com `ocorrenciaId` e exibe dados e fontes retornados.
- [ ] Java/Spring AI apresenta informações oficiais verificadas ou informa quando fontes/localidade não podem ser confirmadas.
- [ ] Tela ADMIN exibe relatórios sem botões de moderação.
- [ ] CRUD está integrado à API real, com erros visuais; seis ou mais telas navegáveis.
- [ ] Tela Sobre mostra hash do commit; build correta no Firebase App Distribution.

## 12. Integração e bloqueios que precisam ser combinados

Antes de ligar definitivamente os serviços, alinhar com o time: nomes dos campos e tipos de ID, issuer/audience/JWT, formato uniforme de erros, paginação, upload, política de edição/exclusão, formato do relatório e contrato de orientação da IA Java, incluindo consulta de ocorrência na API .NET. A equipe do frontend pode começar a desenvolver com os contratos conceituais aqui descritos, mas não deve presumir URLs reais antes de o backend publicar o OpenAPI.
