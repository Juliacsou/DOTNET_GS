# BAIRRO VIVO | Especificação técnica para Java / Spring Boot

**Versão:** 1.1 — 10/11/2026  
**Público:** Desenvolvedor(a) responsável pela disciplina Java Advanced.  
**Objetivo:** Implementar identidade, JWT e a funcionalidade central de IA (orientação cidadã com Spring AI e RAG), interoperando com a API .NET e o Oracle.

> O script Oracle v1 foi criado antes desta revisão e **precisa ser adaptado**. Esta especificação define o comportamento desejado, não confirma o DDL existente. Tipos físicos e contratos serão reconciliados com o time de banco antes de implementar JPA e scripts finais.

## 1. Responsabilidade oficial do Java

**Java é responsável por:**

1. Cadastro, login, JWT, papéis, perfil, senha, preferências e locais favoritos.
2. **Toda a inteligência artificial do aplicativo:** orientação cidadã a partir de uma ocorrência existente, usando Spring AI, LLM e recuperação de fontes oficiais (RAG básico).
3. Base de conhecimento para IA: municípios atendidos, órgãos, serviços, canais de atendimento, horários, documentos e fontes oficiais verificadas, com rastreabilidade. O time define os seeds e manutenção da base; o ADMIN do aplicativo continua apenas consultando relatórios.
4. Histórico de orientações geradas, suas fontes, estado de disponibilidade e metadados técnicos sem dados pessoais desnecessários.
5. Evento assíncrono real via RabbitMQ/Kafka, conforme requisito acadêmico, além de DTOs, validações, segurança, tratamento de erros e OpenAPI.

**.NET é responsável por:** ocorrências e categoria, fotos, localização, comentários, votos, status comunitário, histórico e relatórios. **.NET NÃO implementa IA, prompts, RAG ou chamadas a modelos generativos.**

**Fluxo de orientação:** frontend chama Java com ID da ocorrência -> Java consulta a API .NET para obter a ocorrência e seu município -> Java recupera dados oficiais verificados -> executa RAG com Spring AI -> persiste/retorna orientação e referências -> frontend apresenta a resposta. Java nunca muda o status da ocorrência.

**Arquitetura Java + .NET:** Java emite o JWT; o app o apresenta para ambas as APIs. .NET valida assinatura, claims, expiração e papéis. Java não precisa acessar diretamente tabelas de ocorrências de propriedade .NET: a consulta de dados do problema usa contrato HTTP de leitura.

## 2. Requisitos obrigatórios da disciplina Java Advanced

Conforme o enunciado “2TDS Fevereiro - Global Solution 2026-2.pdf”:

- Spring Boot com API REST de **no mínimo três entidades relacionadas**, CRUD, Spring Data JPA e **paginação**.
- Bean Validation nos DTOs e tratamento global com `@RestControllerAdvice`, com respostas padronizadas.
- Spring Security + JWT com **pelo menos dois papéis** e endpoints protegidos por papel.
- Ao menos **uma funcionalidade real de LLM com Spring AI**, integrada ao fluxo do produto, não chat genérico.
- Arquitetura distribuída: **RabbitMQ/Kafka com publicação e consumo assíncrono** OU pelo menos dois microsserviços com Eureka e comunicação entre eles.
- OpenAPI/Swagger, README para execução e endpoints efetivamente consumidos pelo app Mobile.
- Repositório, apresentação e vídeo conforme enunciado.

**Recomendação:** usar RabbitMQ e Spring AI na mesma aplicação Java, com um produtor de eventos e um consumidor dedicado. A arquitetura atende ao caminho de mensageria sem a sobrecarga de Eureka e dois serviços Java, desde que publicação/consumo assíncronos sejam demonstráveis.

## 3. Domínio Java: três entidades relacionadas

**Nomes conceituais**, sem tipos Oracle definitivos:

| Entidade JPA | Relacionamentos | Campos conceituais | Operações |
|---|---|---|---|
| `Usuario` / futura `TB_USUARIO` | 1:1 com Preferencia; 1:N com LocalFavorito | ID, nome, email, senhaHash, ativo, papel(is), criadoEm, atualizadoEm | Criar, ler própria conta, atualizar, desativar |
| `PreferenciaUsuario` / futura `TB_PREFERENCIA_USUARIO` | 1:1 com Usuario | ID, usuarioId, tema, preferenciasDeAcessibilidade, atualizadoEm | Criar padrão, ler, atualizar, restaurar padrão |
| `LocalFavorito` / futura `TB_LOCAL_FAVORITO` | N:1 com Usuario | ID, usuarioId, apelido, latitude, longitude, criadoEm | Criar, listar paginado, ler, editar, excluir |

A relação entre Usuario e Preferencia é **1:1**. O local favorito é privado e não restringe o cadastro de ocorrência. Não é necessário pedir bairro de residência.

**Entidades adicionais para IA sob responsabilidade Java:** `Municipio`, `OrgaoPublico`, `ServicoPublico`, `CanalAtendimento`, `FonteOficial` e `OrientacaoIA` (e referência de orientação para suas fontes). São conceituais até a revisão do script Oracle; usar os dados de fontes oficiais e orientações sem disputar a responsabilidade do .NET por ocorrências.

Uma quarta entidade opcional, `EventoAuditoria`/`TB_AUDITORIA_EVENTO`, pode registrar eventos da fila para fins de rastreabilidade, **mas deve ser acordada com o banco**. Se não houver tabela de auditoria, o consumidor pode registrar o evento em logging estruturado na demonstração acadêmica.

### Identidade e chaves

- `Usuario.id` deve ser persistente e estável.
- O claim JWT `sub` deve conter uma representação inequívoca desse ID (string). **Escolher UUID ou chave numérica em conjunto com o time Oracle e .NET antes de programar as entidades**.
- O .NET utiliza `sub` para relacionar ocorrências, atualizações e votos ao autor autenticado. **Nunca confiar em um `idUsuario` enviado pelo corpo da requisição**.
- A API .NET não deve receber nem consultar o hash da senha. Se Java e .NET compartilharem o mesmo Oracle, separar permissões de banco por responsabilidade.
- No caso de anonimato público, o identificador real continua vinculado ao registro internamente, mas as respostas públicas não expõem sua identidade.

## 4. Papéis e regras de segurança

Papéis iniciais: `CIDADAO` e `ADMIN`.

- `CIDADAO`: perfil, favoritos, preferências, solicitação de orientação cidadã com IA e participação nas ocorrências via .NET.
- `ADMIN`: relatórios/indicadores via .NET. **Não concede autorização para registrar, atualizar ou moderar ocorrências.** No MVP, o papel ADMIN não se acumula com CIDADAO; a conta de administrador é exclusivamente de consulta e relatórios.
- Visitante: não precisa token para GET público do .NET. **Não existe “conta visitante” nem JWT anônimo**.
- Cadastro público sempre produz papel `CIDADAO`. **Nunca aceitar `ADMIN` fornecido pelo próprio cliente**.
- Provisionamento de ADMIN feito via seed/rotina segura operada pela equipe, não via cadastro aberto.

### JWT recomendado

Assinatura **RS256** (chave privada fica com Java; .NET recebe chave pública, idealmente via endpoint JWKS com rotação controlada). Claims:

```json
{
  "iss": "bairro-vivo-java",
  "aud": "bairro-vivo-apis",
  "sub": "123",
  "roles": ["CIDADAO"],
  "iat": 1794319200,
  "exp": 1794322800
}
```

**Somente exemplo de formato, não token real.** Os valores numéricos e os nomes de issuer/audience serão combinados com .NET. Usar tempo de expiração curto; ex.: 60 min como proposta. O cliente usa `Authorization: Bearer <jwt>` nas ações privadas.

**Importante:** frontend e .NET precisam concordar em: nome do claim de papéis (`roles`), tipo de `sub`, emissor, audiência, algoritmo, expiração e distribuição da chave pública.

### Logout e recuperação

- MVP: logout remove JWT do aplicativo; a validade do token já emitido termina naturalmente na expiração. Não prometer revogação instantânea sem infraestrutura para isso.
- Fluxo de refresh token é melhoria recomendada, mas não deve ser presumido se não implementado. Se adotado, usar refresh token rotativo e revogação explícita.
- Alterar senha: requer senha atual ou fluxo seguro de recuperação.
- Recuperar senha via email exige provedor de envio, token de uso único e expiração; se não concluído até a entrega, não expor tela prometendo recuperação funcional.
- Nunca armazenar senha em texto claro; utilizar BCrypt/Argon2 ou equivalente adequado, com sal.

## 5. Endpoints sugeridos — Java

São **propostas de contrato**, não endpoints implementados. Prefixo ilustrativo: `/api/v1`.

### 5.1 Autenticação e conta

| Método + rota sugerida | Acesso | Função / observações |
|---|---|---|
| `POST /auth/registrar` | Público | Criar `CIDADAO`; email único; retorna perfil mínimo e/ou token conforme contrato combinado |
| `POST /auth/login` | Público | Validar credenciais e retornar JWT + perfil mínimo |
| `POST /auth/logout` | Autenticado | Opcional no MVP sem refresh/revogação; logout local no app continua obrigatório |
| `GET /usuarios/me` | Autenticado | Obter dados da própria conta |
| `PATCH /usuarios/me` | Autenticado | Atualizar nome e outros dados permitidos |
| `DELETE /usuarios/me` | Autenticado | Desativação/solicitação de exclusão, com política de dados acordada |
| `PUT /usuarios/me/senha` | Autenticado | Alterar senha com validação da senha atual |
| `POST /auth/recuperar-senha` | Público | P1, somente se email/transporte seguro estiverem funcionando |
| `POST /auth/redefinir-senha` | Público + token de recuperação | P1, confirmar token único e expirável |

### 5.2 Preferências e locais favoritos

| Método + rota sugerida | Acesso | Função / observações |
|---|---|---|
| `GET /usuarios/me/preferencias` | Autenticado | Consultar preferências do próprio usuário |
| `PUT /usuarios/me/preferencias` | Autenticado | Atualizar/criar preferências (upsert controlado) |
| `DELETE /usuarios/me/preferencias` | Autenticado | Restaurar padrão; não apagar conta |
| `GET /usuarios/me/locais-favoritos?page=0&size=10` | Autenticado | Listar com paginação Spring Data |
| `POST /usuarios/me/locais-favoritos` | Autenticado | Cadastrar local com nome + coordenadas |
| `GET /usuarios/me/locais-favoritos/{id}` | Dono | Consultar um local favorito |
| `PUT /usuarios/me/locais-favoritos/{id}` | Dono | Editar apelido e/ou coordenadas |
| `DELETE /usuarios/me/locais-favoritos/{id}` | Dono | Excluir local favorito |

A paginação deve ser real (por exemplo, `Pageable`/`Page<T>`), retornando metadados previsíveis de página, total e quantidade. Preferências são 1:1; portanto a paginação aplica-se de maneira natural a `LocalFavorito`.

### 5.3 IA de orientação cidadã (Spring AI + RAG)

**Esta é a única funcionalidade de IA do MVP.** Não implementar o endpoint antigo `POST /ia/melhorar-descricao` (descontinuado, não construir).

| Método e rota sugeridos | Acesso | Função / observações |
|---|---|---|
| `POST /ia/orientacoes` | `CIDADAO` | Solicitar nova orientação informando apenas o ID da ocorrência existente |
| `GET /ia/orientacoes/ocorrencias/{ocorrenciaId}` | Público | Listar orientações publicadas e filtradas, sem dados pessoais do solicitante |
| `GET /ia/orientacoes/{id}` | Público se publicável; de outro modo, apenas solicitante | Obter resposta ou estado de processamento, respeitando visibilidade |
| `GET /ia/orientacoes/minhas` | `CIDADAO` (P1) | Histórico pessoal, sem acesso a dados de outros solicitantes |

**Requisição conceitual:**

```json
{
  "ocorrenciaId": "101"
}
```

**Resposta conceitual:**

```json
{
  "id": "501",
  "ocorrenciaId": "101",
  "estado": "DISPONIVEL",
  "municipio": { "nome": "São Paulo", "uf": "SP" },
  "orgaoResponsavel": "Servico municipal pertinente (validar na fonte)",
  "servico": "Atendimento a ocorrencias urbanas",
  "canais": [
    { "tipo": "SITE", "valor": "https://example.org/placeholder", "horario": null }
  ],
  "passos": ["Verifique os requisitos no canal oficial indicado."],
  "cuidados": ["Evite se aproximar de locais com risco aparente."],
  "fontes": [
    { "titulo": "Pagina oficial de demonstracao", "url": "https://example.org/placeholder", "verificadaEm": "2026-11-10" }
  ],
  "geradaEm": "2026-11-10T22:00:00Z"
}
```

**Observação:** dados e URLs acima são apenas ilustrativos, não referências verificadas; NUNCA utilizar esses valores como resposta real de IA.

**Processamento no backend Java:**

1. Validar JWT (`CIDADAO`), limite de requisições e ID; extrair `sub` como solicitante. Não aceitar identidade, prompt ou município arbitrários enviados pelo cliente.
2. Buscar via API .NET **dados canônicos e públicos** da ocorrência: ID, título/descrição, categoria, coordenadas e município identificado com confiabilidade. Se necessário, obter contexto mínimo de localização por contrato interno autenticado.
3. Verificar município suportado. MVP: base oficial do município de São Paulo, se a localização efetiva indicar essa cidade. Ausência de município confiável => `LOCALIDADE_NAO_CONFIRMADA`.
4. Recuperar documentos/trechos de **fontes oficiais previamente coletadas e verificadas**, filtrando por município e tipo de problema. Documentar ingestão, recuperação de trechos e associação das fontes a resposta. Aplicar RAG básico antes de montar o prompt para o modelo.
5. Usar Spring AI para gerar resposta estruturada apenas com o contexto recuperado. Canais, telefones e horários sem fonte => campos nulos ou ausentes; nenhum dado inventado. A resposta explicita incertezas e atribuidade ambígua.
6. Salvar metadados da consulta, estado de retorno, versão do modelo e fontes citadas. Não registrar PII sensível ou prompts completos por padrão. Publicar somente resposta revisada para ausência de dados pessoais.
7. Entregar ao frontend órgão, serviço, canais, horários, passos, avisos, fontes com links e data de verificação. Se não houver fonte pertinente, responder `SEM_FONTE` ou `PARCIAL` (nunca preencher contatos por suposição).

**Estados de negócio propostos:** `DISPONIVEL`, `PARCIAL`, `SEM_FONTE`, `LOCALIDADE_NAO_CONFIRMADA`, `ERRO`. Formato final deve ser acordado com frontend. Não chamar de 'serviço oficial do Bairro Vivo' nem afirmar que um protocolo foi criado; o sistema apenas orienta a pessoa a procurar os canais adequados.

**Segurança:** usar timeouts, rate limiting e limites de custo; normalizar texto de relatos como dado não confiável (proteger contra prompt injection). Validar links externos, usar allowlist de fontes oficiais selecionadas e não executar instruções recuperadas. Em risco imediato, orientar a não se expor ao perigo e procurar atendimento competente, sem instruções operacionais perigosas.

**Testes e avaliação para Java e Disruptive Architectures:** ao menos exemplos de problemas e municípios suportados/não suportados, falta de contato, falta de horário, fontes conflitantes, ocorrência inexistente, prompt injection no relato, LLM indisponivel e rastreabilidade de fontes. Medir pelo menos adequação da competência do órgão, fidelidade aos documentos e ausência de contatos inventados.

### 5.4 Possível endpoint de chave pública

| Método + rota sugerida | Acesso | Uso |
|---|---|---|
| `GET /.well-known/jwks.json` | Público | Publicar chaves públicas para validação de JWT pelo .NET (se a equipe adotar JWKS) |

Caso o grupo prefira distribuir a chave pública via configuração segura, o endpoint JWKS não é obrigatório no MVP. **Não expor chave privada em nenhum cenário.**

## 6. Formatos de request/response a padronizar

### Cadastro

```json
{
  "nome": "Maria Silva",
  "email": "maria@example.com",
  "senha": "exemplo-de-senha-segura",
  "confirmacaoSenha": "exemplo-de-senha-segura"
}
```

`confirmacaoSenha` pode ser validado somente no frontend e não precisa ser persistido. Não incluir papel ou privilégios no cadastro público.

### Login

```json
{
  "email": "maria@example.com",
  "senha": "exemplo-de-senha-segura"
}
```

**Resposta sugerida:**

```json
{
  "accessToken": "<jwt>",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "usuario": {
    "id": "123",
    "nome": "Maria Silva",
    "email": "maria@example.com",
    "papeis": ["CIDADAO"]
  }
}
```

### Local favorito

```json
{
  "apelido": "Faculdade",
  "latitude": -23.5600000,
  "longitude": -46.6500000
}
```

### Resposta de erro uniforme

```json
{
  "codigo": "VALIDACAO_INVALIDA",
  "mensagem": "Confira os dados informados.",
  "detalhes": [
    { "campo": "email", "mensagem": "Informe um email válido." }
  ],
  "correlationId": "exemplo-correlation-id"
}
```

**Padrão sugerido:** `400` campos inválidos, `401` credenciais/token inválidos, `403` papel insuficiente, `404` dado inexistente/de outro dono, `409` email já existente, `429` excesso de requisições, `500` falha inesperada sem stack trace pública. Diferenciar `401` de `403` para o frontend.

## 7. Validações JPA/Bean Validation e integridade

- `Usuario`: nome não vazio, tamanho máximo acordado; email normalizado/único; senha com política mínima de tamanho; ativo e timestamp no servidor.
- `PreferenciaUsuario`: um registro por usuário; regras de enum ou lista controlada; sem dados residenciais obrigatórios.
- `LocalFavorito`: apelido não vazio; latitude `[-90,90]`, longitude `[-180,180]`; usuário obtido do JWT; impedir acesso aos favoritos de terceiros.
- DTOs independentes das entidades. **Nunca retornar `senhaHash`**.
- `@RestControllerAdvice` para `MethodArgumentNotValidException`, autenticação, conflito, recurso ausente e falha do modelo, da recuperação de fontes ou da API .NET.
- JPA: projetar índices para email e usuário nos locais favoritos; tratamento de n+1 queries e paginação em consultas.
- O Oracle v1 existente e preliminar e precisa ser reconciliado com está divisao de IA; revisar chaves, índices, tabelas de conhecimento e orientação antes do mapeamento JPA.

## 8. Mensageria distribuída — proposta RabbitMQ

Para atender à opção A do requisito de Java, sugerimos:

```text
POST /auth/registrar
  -> validação / persistência do Usuario
  -> publicação do evento usuario.cadastrado
  -> RabbitMQ (exchange/queue)
  -> consumidor assíncrono Java
  -> registro estruturado de auditoria e/ou criação de preferências iniciais
```

**Contrato conceitual do evento:**

```json
{
  "eventId": "uuid-do-evento",
  "type": "usuario.cadastrado",
  "versao": 1,
  "occurredAt": "2026-11-10T20:00:00Z",
  "usuarioId": "123"
}
```

- **Não publicar email, senha, hash ou token** nos eventos.
- Consumidor precisa receber efetivamente a mensagem, executar ação observável e ser idempotente (`eventId`).
- Preferir publicação após commit da transação; documentar a limitação de entrega caso não haja outbox transacional.
- Demonstrar a fila, publicação e consumo durante a entrega; não basta importar uma dependência RabbitMQ.
- Se optar por criar preferências iniciais via consumidor, o GET de preferências deve tolerar o pequeno intervalo entre cadastro e processamento; alternativa é manter criação síncrona e usar evento apenas para auditoria.
- O RabbitMQ e o consumidor devem estar disponíveis no ambiente de demonstração; prever tratamento de indisponibilidade.

## 9. Integração com .NET e Oracle

### Contrato de confiança

1. Java assina JWT; o .NET valida `iss`, `aud`, assinatura, `exp` e `roles`.
2. `sub` é o ID canônico do usuário. O .NET o registra como autor de ocorrência, atualização e voto.
3. Oracle deve suportar consistência desse ID em ambos os domínios. A escolha do tipo exato será fechada **antes do SQL**.
4. O .NET não faz chamadas de login/senha ao Java para cada ação; valida a assinatura localmente usando chave pública.
5. Se .NET precisar exibir o nome de um autor não anônimo, pode obter somente os campos públicos autorizados por consulta de leitura ou serviço Java; **não acessar credenciais sensíveis**.
6. Para publicações anônimas, o backend .NET retorna `autorExibicao = "Cidadão anônimo"` e **não retorna o ID real do autor em GET público**.
7. Todas as checagens de dono usam o JWT validado; papel ADMIN não permite alterar status por conta própria.

8. Para gerar orientação, **Java chama a API .NET somente para leitura dos dados da ocorrência** (ID, título, descrição, categoria, município, coordenadas se necessário). O frontend não precisa orquestrar a comunicação entre APIs.
9. A API .NET não acessa modelo generativo nem precisa servir endpoints de IA. A persistencia dos resultados de orientação e documentos de fonte pertence ao dominio Java, mesmo usando o mesmo Oracle.
10. Os endpoints públicos de orientação retornam apenas informações publicaveis, sem `solicitanteId` nem identidade privada. O Java deve validar existencia e visibilidade de cada registro.
11. Se o município não puder ser confirmado no contrato de leitura, Java não deve supor que a ocorrência fica no município do perfil do usuário.


### Política conjunta sobre desativação/exclusão de conta

Para não quebrar o histórico comunitário, a desativação/exclusão de conta precisa ser coordenada: definir quais dados pessoais serão eliminados/anônimos e quais registros de interesse público poderão permanecer sem identificação pessoal. Isso precisa ser tratado no modelo final e na política de privacidade; **não implementar `DELETE CASCADE` de usuário para toda a base de ocorrências** sem essa decisão.

### Configuração segura

Separar `.env` / secrets de dev e produção: conexão Oracle, chave privada JWT, modelo Spring AI, RabbitMQ e URLs dos serviços. Não commitar chaves, senhas nem credenciais no repositório. Padronizar CORS para as origens aprovadas e HTTPS no deploy.

## 10. Testes recomendados

| Cenário | Resultado esperado |
|---|---|
| Cadastro válido | `201` ou `200` acordado; cria CIDADAO e não expõe senha |
| Email repetido | `409`; não cria usuário duplicado |
| Login correto/incorreto | JWT válido / `401` sem vazamento de informações |
| Token inválido/expirado | `401` em endpoints privados |
| CIDADÃO tenta chamar endpoint ADMIN | `403` |
| Cliente envia `ADMIN` no cadastro | Ignorado/rejeitado; cadastro mantém CIDADAO |
| Leitura/edição de favorito de outra conta | `404` ou `403` definido em contrato; sem vazamento |
| Favoritos com página 0 e tamanho 10 | Conteúdo e metadados de paginação coerentes |
| IA orienta ocorrência com fonte oficial | Resposta estruturada cita a fonte e não altera ocorrência |
| IA indisponível | Erro padronizado; frontend informa indisponibilidade sem inventar contato |
| Evento `usuario.cadastrado` | Publicado, consumido e verificável |
| Mesmo `eventId` consumido novamente | Consumidor não duplica efeito |
| Município sem fonte confirmada | `SEM_FONTE` ou `LOCALIDADE_NAO_CONFIRMADA`, sem telefone inventado |
| Dados da ocorrência recuperados pela API .NET | Java utiliza ID válido e somente leitura; não le tabelas .NET diretamente |
| Prompt injection no relato | Java ignora instruções do texto da ocorrência e limita resposta as fontes oficiais |
| JWT emitido em Java aceito pelo .NET | Integração real, sem novo login |

## 11. Critérios de entrega Java

- [ ] API Spring Boot executável, conectada ao Oracle fornecido pelo grupo.
- [ ] Três entidades JPA relacionadas, CRUD e uma consulta paginada real.
- [ ] DTOs com Bean Validation e tratamento global de erros.
- [ ] Spring Security + JWT emitido por Java, papéis CIDADAO e ADMIN e autorização demonstrável.
- [ ] JWT reconhecido pelo .NET, com contrato fechado e sem compartilhamento de senha.
- [ ] Spring AI/RAG integrado à tela “Como resolver este problema?”, recuperando ocorrência via .NET e fontes oficiais sob responsabilidade Java.
- [ ] Evento publicado e consumido assincronamente via RabbitMQ, com evidência.
- [ ] Swagger/OpenAPI e README com configuração, execução, exemplos, usuário de teste e declaração de uso de IA.
- [ ] Testes automatizados e demonstração ponta a ponta no aplicativo.

## 12. Pendências para alinhamento com a próxima etapa (banco de dados)

**Não gerar novo SQL nesta etapa.** Já existe um SQL Oracle v1 preliminar, anterior a esta decisão. Antes de alinhar JPA e atualizar o SQL, confirmar:

1. Tipo da chave de usuário e representação no `sub` (NUMBER/UUID).
2. Nomes e tipos físicos definitivos das tabelas/colunas Oracle.
3. Representação do papel em campo único no MVP e eventual expansão futura para múltiplos papéis.
4. Regras de exclusão/desativação de usuário, privacidade e histórico.
5. Critério de criação de PreferenciaUsuario (síncrono ou por evento).
6. Algoritmo/chaves JWT, issuer, audience, claim `roles`, expiração e estratégia de renovação.
7. Limites de requisições, timeout e custo da orientação com Spring AI, formatação das fontes e município piloto.
8. Endpoint de leitura de ocorrência .NET consumido por Java e contrato de resposta Java para o frontend, ambos em OpenAPI.
9. Tabelas/relacoes de conhecimento oficial, versões de fonte e registro de orientação no Oracle, de propriedade Java.

**Regra de governança entre equipes:** alterações nesses contratos precisam ser combinadas com frontend, .NET e banco, pois todos dependerão da mesma identidade de usuário.
