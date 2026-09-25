# IAM Module — Guia de Integração

> Este guia é destinado aos desenvolvedores da API Backend do **Sistema de Gestão de XXXX** que vão **consumir** o módulo IAM como dependência Maven no projeto Spring Boot. O módulo resolve os requisitos de usuários/perfis de acesso e trilha de auditoria do projeto — ver [`../requisitos/requirements.md`](../requisitos/requirements.md).

---

## 1. Pré-requisitos

- Java 17+
- Spring Boot 3.2.4+
- Banco de dados relacional (H2 para desenvolvimento, PostgreSQL para produção — ver [`architecture.md`](./architecture.md))
- Maven

---

## 2. Instalação

### 2.1 Adicionar dependência

No `pom.xml` da API Backend:

```xml
<dependency>
    <groupId>br.com.cati</groupId>
    <artifactId>iam-module</artifactId>
    <version>1.0.0</version>
</dependency>
```

### 2.2 Garantir que o Spring escaneia o pacote do IAM

Na classe principal da aplicação:

```java
@SpringBootApplication(scanBasePackages = {
    "<pacote-base-do-projeto>", // pacote do sistema
    "br.com.cati.iam"           // modulo IAM
})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

> O IAM usa `@AutoConfiguration`, entao em muitos casos o scan e automatico. Mas declarar explicitamente garante que nao havera problemas.

### 2.3 Configurar banco de dados

O IAM cria as tabelas automaticamente via JPA. Configure o datasource no `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:h2:file:./data/db                       # desenvolvimento
    # url: jdbc:postgresql://localhost:5432/xxxx      # producao

  jpa:
    hibernate:
      ddl-auto: update       # desenvolvimento (usar 'validate' em producao)
    show-sql: false
```

> O IAM gerencia seu próprio schema (`iam_*`) via `ddl-auto` da biblioteca — independente do Flyway que versiona o schema dos módulos de negócio. Ver [`architecture.md — Banco de dados, perfis e migrações`](./architecture.md#banco-de-dados-perfis-e-migrações).

**Tabelas criadas pelo módulo:**

| Tabela                     | Descricao                                    |
|---------------------------|----------------------------------------------|
| `iam_users`               | Usuários do sistema (todos os perfis)        |
| `iam_opaque_tokens`       | Tokens de autenticacao                       |
| `iam_confirmation_tokens` | Tokens de confirmacao (reset de senha)       |
| `iam_roles`               | Roles do RBAC                                |
| `iam_permissions`          | Permissions do RBAC                          |
| `iam_role_permissions`    | Relacao N:N roles <-> permissions             |
| `iam_user_context_roles`  | Atribuicao de roles a usuarios por contexto  |
| `iam_audit_log`           | Log de auditoria (atende RF06/RNF05)          |

---

## 3. Configuração

Todas as configurações têm valores default e podem ser sobrescritas no `application.yml`:

```yaml
iam:
  auth:
    expiration-minutes: 30                      # duracao do token
    sliding-threshold-minutes: 10               # renovacao automatica do token
    max-login-attempts: 5                       # tentativas antes de bloquear
    lockout-duration-minutes: 30                # duracao do bloqueio
    verification-email-expiration-hours: 24     # expiracao link de confirmacao
    password-reset-expiration-hours: 1          # expiracao link de reset
    max-active-sessions: 5                      # sessoes ativas por usuario (0 = sem limite)

  argon2:
    salt-length: 16
    hash-length: 32
    parallelism: 1
    memory: 16384       # 16 MB
    iterations: 2
```

Se você não declarar nada, os defaults acima serão usados automaticamente.

---

## 4. Fluxos de uso

### 4.1 Criar usuário

> Se o projeto não tiver cadastro público, toda conta é criada por um Administrador — este endpoint deve ficar protegido, não em `permitAll` (ver [`architecture.md`](./architecture.md#autenticação-e-perfis-de-acesso-iam)):

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/v1/users")
public class UserController {

    private final IamCreateIamUserUseCase createUserUseCase;

    @PostMapping
    @PreAuthorize("hasAuthority('ROLE_ADMINISTRADOR')")
    public ResponseEntity<?> criar(@RequestBody CriarUsuarioRequest request) {
        var result = createUserUseCase.execute(
            new IamCreateIamUserCommand(request.email(), request.senha())
        );

        // result contem:
        //   - result.userId()            → UUID do usuario criado
        //   - result.email()             → email do usuario
        //   - result.confirmationToken() → token para confirmacao de email, se aplicavel

        return ResponseEntity.status(201).body(Map.of(
            "userId", result.userId(),
            "email", result.email()
        ));
    }
}
```

Após criar o usuário, atribua o perfil correspondente (ver seção 5.2/5.3) usando as roles definidas para o projeto.

**Requisitos de senha:** O módulo valida senhas com a anotação `@IamStrongPassword`. A senha deve ter:
- Mínimo de 8 caracteres
- Pelo menos uma letra maiúscula
- Pelo menos uma letra minúscula
- Pelo menos um dígito
- Pelo menos um caractere especial

Se a senha não atender aos critérios, uma `ConstraintViolationException` será lançada. Essa validação se aplica à criação de usuário, alteração de senha e reset de senha.

**Nota de segurança:** As mensagens de exceção do módulo são intencionalmente genéricas para prevenir vazamento de informações. Por exemplo, `IamEmailAlreadyExistsException` retorna "Não foi possível completar o cadastro" (sem revelar se o email já existe). Não exponha detalhes internos nas respostas HTTP.

### 4.2 Login (autenticação)

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/v1/auth")
public class AuthController {

    private final IamAuthenticateUseCase authenticateUseCase;

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        try {
            var result = authenticateUseCase.execute(
                new IamAuthenticateCommand(request.email(), request.senha())
            );

            // result.token()     → token opaco para o cliente usar
            // result.principal() → dados do usuario autenticado (com roles)

            return ResponseEntity.ok(Map.of(
                "token", result.token(),
                "userId", result.principal().id(),
                "email", result.principal().email()
            ));
        } catch (IamUnauthorizedException e) {
            return ResponseEntity.status(401).body("Credenciais invalidas");
        } catch (IamAccountLockedException e) {
            return ResponseEntity.status(423).body("Conta temporariamente bloqueada");
        }
    }
}
```

**Fluxo interno do login:**
1. Busca usuário por email
2. Verifica se a conta está bloqueada (lockout)
3. Valida a senha com Argon2
4. Gera token opaco (SecureRandom, 256 bits) e armazena hash SHA-256 no banco
5. Remove sessões excedentes se `max-active-sessions` for atingido (FIFO — sessões mais antigas são removidas primeiro)
6. Publica evento de auditoria (RF06)
7. Retorna token + principal

### 4.3 Usar o token nas requisições

O cliente (SPA React) deve enviar o token no header `Authorization`:

```
Authorization: Bearer <token>
```

O `IamTokenAuthFilter` intercepta automaticamente todas as requisições e:
1. Extrai o token do header
2. Valida no banco de dados
3. Resolve roles e permissions
4. Popula `IamContext` e `SecurityContextHolder`

Você não precisa fazer nada — o filtro já está registrado automaticamente.

### 4.4 Acessar o usuário autenticado

Em qualquer parte do código (controllers, usecases), use `IamContext`:

```java
// Obter o usuario autenticado (lanca exception se nao autenticado)
Principal principal = IamContext.getPrincipal();
UUID userId = principal.id();
String email = principal.email();

// Verificar permissoes
if (principal.hasPermission("RECURSO_CRIAR")) {
    // ...
}

if (principal.hasRole("ROLE_ADMINISTRADOR")) {
    // ...
}

if (principal.hasAnyPermission("RECURSO_GERENCIAR", "RECURSO_CONSULTAR")) {
    // ...
}

// Opcional — nao lanca exception se nao autenticado
Optional<Principal> optional = IamContext.getPrincipalOrEmpty();
boolean autenticado = IamContext.isAuthenticated();
```

Se o projeto tiver uma regra de escopo de dados específica do domínio (ex.: um usuário só enxerga os dados de uma entidade à qual está vinculado), combine `IamContext` com um resolver de domínio próprio do módulo responsável — ver a seção de decisões de arquitetura específicas do domínio em [`architecture.md`](./architecture.md). O IAM resolve *quem* está autenticado; o escopo de *quais dados* ele vê é regra de negócio do domínio, fora do IAM.

### 4.5 Configurar Spring Security

Configure quais endpoints são públicos e quais exigem autenticação:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
                                            IamTokenAuthFilter tokenFilter) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/v1/auth/login",
                    "/v1/confirmation/**"
                ).permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(tokenFilter,
                UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

Se o projeto não tiver cadastro público, `/v1/users` **não** deve estar na lista de `permitAll` — a criação de usuário exige `ROLE_ADMINISTRADOR` (ver seção 4.1).

### 4.6 Proteger endpoints com permissões

Use `@PreAuthorize` do Spring Security. Exemplos aplicados aos módulos do sistema (ver [`architecture.md`](./architecture.md)):

```java
@PostMapping("/<recurso>")
@PreAuthorize("hasAuthority('RECURSO_CRIAR')")               // RF0X — nome do requisito
public ResponseEntity<?> criarRecurso(@RequestBody RecursoRequestDTO request) {
    // ...
}

@GetMapping("/<recurso>/{id}")
@PreAuthorize("hasAuthority('RECURSO_CONSULTAR')")           // RF0X — nome do requisito
public RecursoResponseDTO consultarRecurso(@PathVariable UUID id) {
    // ...
}
```

As roles e permissions resolvidas pelo IAM são registradas como `GrantedAuthority` no Spring Security, então `hasAuthority()` funciona naturalmente.

### 4.7 Logout

```java
@PostMapping("/logout")
public ResponseEntity<?> logout(@RequestHeader("Authorization") String header) {
    String token = header.replace("Bearer ", "");
    logoutUseCase.execute(token);
    return ResponseEntity.ok().build();
}

// Logout de todos os dispositivos
@PostMapping("/logout-all")
public ResponseEntity<?> logoutAll() {
    var principal = IamContext.getPrincipal();
    logoutAllDevicesUseCase.execute(principal.id());
    return ResponseEntity.ok().build();
}
```

### 4.8 Alterar senha

Requer que o usuário esteja autenticado:

```java
@PutMapping("/change-password")
public ResponseEntity<?> alterarSenha(@RequestBody AlterarSenhaRequest request) {
    var result = changePasswordUseCase.execute(
        new IamChangePasswordCommand(request.senhaAtual(), request.novaSenha())
    );
    // Apos alterar a senha, TODAS as sessoes ativas sao invalidadas automaticamente.
    // O usuario precisara fazer login novamente.
    return ResponseEntity.ok().build();
}
```

### 4.9 Reset de senha (esqueci minha senha)

Fluxo em 2 etapas:

**Etapa 1 — Gerar token de reset:**

```java
@PostMapping("/forgot-password")
public ResponseEntity<?> esqueciSenha(@RequestBody EsqueciSenhaRequest request) {
    var user = userRepository.findByEmail(request.email())
        .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

    var token = generateConfirmationUseCase.execute(
        new IamGenerateConfirmationCommand(user.getId(), ConfirmationType.PASSWORD_RESET)
    );

    // VOCE envia o email com o token
    enviarEmailReset(request.email(), token.value());

    return ResponseEntity.ok("Email de recuperacao enviado");
}
```

**Etapa 2 — Resetar com o token:**

```java
@PostMapping("/reset-password")
public ResponseEntity<?> resetarSenha(@RequestBody ResetarSenhaRequest request) {
    resetPasswordUseCase.execute(
        new IamResetPasswordCommand(request.token(), request.novaSenha())
    );
    // Todas as sessoes ativas sao invalidadas automaticamente.
    return ResponseEntity.ok("Senha alterada com sucesso");
}
```

---

## 5. RBAC — Roles e Permissions do sistema

### 5.1 Conceitos

- **Role:** perfil do usuário no sistema. Deve começar com `ROLE_`.
- **Permission:** ação específica sobre um módulo de negócio. Uppercase livre.
- **Contexto:** um UUID que representaria um escopo (ex.: outra instituição, outra unidade). Decida se o projeto usa esse recurso ou se todas as roles são atribuídas globalmente (contexto `null`) — registre a decisão em [`architecture.md`](./architecture.md). Se o escopo de dados de um perfil depender de uma regra de domínio (não de um contexto arbitrário), resolva-o no módulo de negócio responsável, não pelo contexto do IAM.

### 5.2 Roles do sistema

> Preencha com os perfis definidos nos requisitos ([`../requisitos/requirements.md`](../requisitos/requirements.md)).

| Role | Perfil de negócio |
|---|---|
| `ROLE_ADMINISTRADOR` | Administrador do sistema |
| `ROLE_...` | |

### 5.3 Permissions por módulo

> Uma permission de escrita e uma de consulta por módulo é um bom ponto de partida; adicione outras conforme a granularidade que o módulo exigir.

| Módulo | Permission | Requisito |
|---|---|---|
| `<modulo1>` | `<MODULO1>_CRIAR`, `<MODULO1>_CONSULTAR` | RF01 |
| `<modulo2>` | `<MODULO2>_GERENCIAR`, `<MODULO2>_CONSULTAR` | RF02 |

### 5.4 Permissions sugeridas por role

Mapeamento de negócio (seed inicial) — perfis somente-leitura nunca recebem uma permission de escrita:

| Role | Permissions |
|---|---|
| `ROLE_ADMINISTRADOR` | Todas as permissions de negócio + gestão de usuários |
| `ROLE_...` | |

### 5.5 Criar roles e permissions (seed inicial)

```java
// Criar roles
createRoleUseCase.execute(new IamCreateRoleCommand("ROLE_ADMINISTRADOR"));
createRoleUseCase.execute(new IamCreateRoleCommand("ROLE_USUARIO"));

// Criar permissions
createPermissionUseCase.execute(new IamCreatePermissionCommand("RECURSO_CRIAR"));
createPermissionUseCase.execute(new IamCreatePermissionCommand("RECURSO_CONSULTAR"));

// Vincular permissions a role
assignPermissionToRoleUseCase.execute(
    new IamAssignPermissionToRoleCommand("ROLE_USUARIO", "RECURSO_CRIAR")
);
assignPermissionToRoleUseCase.execute(
    new IamAssignPermissionToRoleCommand("ROLE_ADMINISTRADOR", "RECURSO_CONSULTAR")
);
```

### 5.6 Atribuir roles a usuários

```java
// Role global (sem contexto) ou escopada, dependendo da decisão em 5.1
assignRoleToUserUseCase.execute(
    new IamAssignRoleCommand(userId, "ROLE_USUARIO", null)
);
```

### 5.7 Revogar roles e permissions

```java
// Revogar role de usuario
revokeRoleFromUserUseCase.execute(
    new IamRevokeRoleCommand(userId, "ROLE_USUARIO", null)
);

// Revogar permission de role
revokePermissionFromRoleUseCase.execute(
    new IamRevokePermissionFromRoleCommand("ROLE_USUARIO", "RECURSO_CRIAR")
);
```

---

## 6. Exceptions do módulo

O módulo lança exceptions específicas que você deve tratar nos seus controllers:

| Exception                        | Quando                                      | HTTP sugerido |
|---------------------------------|----------------------------------------------|---------------|
| `IamUnauthorizedException`      | Credenciais invalidas ou usuario nao autenticado | 401       |
| `IamAccountLockedException`     | Conta bloqueada por tentativas excessivas    | 423           |
| `IamEmailAlreadyExistsException`| Cadastro nao concluido (mensagem generica por seguranca) | 409  |
| `IamForbiddenException`         | Sem permissao para a acao                    | 403           |
| `IamTokenNotFoundException`     | Token nao encontrado no banco                | 404           |
| `IamTokenExpiredException`      | Token expirado                               | 410           |
| `IamTokenAlreadyUsedException`  | Token de confirmacao ja utilizado             | 410           |

Recomendação — crie um `@ControllerAdvice` para tratar todas:

```java
@RestControllerAdvice
public class IamExceptionHandler {

    @ExceptionHandler(IamUnauthorizedException.class)
    public ResponseEntity<?> unauthorized(IamUnauthorizedException e) {
        return ResponseEntity.status(401).body(Map.of("error", e.getMessage()));
    }

    @ExceptionHandler(IamAccountLockedException.class)
    public ResponseEntity<?> locked(IamAccountLockedException e) {
        // NAO exponha e.getLockoutUntil() ao cliente — informacao de timing pode ser explorada
        return ResponseEntity.status(423).body(Map.of(
            "error", "Conta temporariamente bloqueada"
        ));
    }

    @ExceptionHandler(IamEmailAlreadyExistsException.class)
    public ResponseEntity<?> emailExists(IamEmailAlreadyExistsException e) {
        return ResponseEntity.status(409).body(Map.of("error", e.getMessage()));
    }

    @ExceptionHandler(IamForbiddenException.class)
    public ResponseEntity<?> forbidden(IamForbiddenException e) {
        return ResponseEntity.status(403).body(Map.of("error", e.getMessage()));
    }

    @ExceptionHandler({IamTokenNotFoundException.class, IamTokenExpiredException.class,
                        IamTokenAlreadyUsedException.class})
    public ResponseEntity<?> tokenError(RuntimeException e) {
        return ResponseEntity.status(410).body(Map.of("error", e.getMessage()));
    }
}
```

---

## 7. Auditoria (RF06 / RNF05)

O módulo registra automaticamente eventos de segurança na tabela `iam_audit_log`. Você não precisa fazer nada — é automático e assíncrono.

> **Ponto em aberto:** se o projeto tiver um requisito de auditabilidade estrita (todo dado sensível rastreável até o usuário e o momento exato do registro), decida se o registro do log de auditoria precisa bloquear/reverter a operação original em caso de falha — uma garantia síncrona que o comportamento assíncrono descrito acima não oferece hoje.

Eventos registrados automaticamente:
- Login sucesso/falha
- Conta bloqueada
- Logout (único e todos os dispositivos)
- Senha alterada/resetada
- Conta criada
- Roles atribuídas/revogadas
- Permissions atribuídas/revogadas

Para registrar eventos customizados de negócio do sistema, use o `IamRegisterAuditEventUseCase`:

```java
@RequiredArgsConstructor
public class RecursoUseCase {
    private final IamRegisterAuditEventUseCase registerAuditUseCase;

    public void criar(UUID recursoId, String detalhe) {
        // ... logica de negócio do use case ...

        var principal = IamContext.getPrincipal();
        registerAuditUseCase.execute(
            "RECURSO_CRIADO",
            principal.id(),
            IpAddressUtil.extractIpAddress(),
            null,
            "Recurso #" + recursoId + " criado: " + detalhe
        );
    }
}
```

Para consultar o audit log, injete o `IamAuditLogRepository` no seu código:

```java
@Autowired
private IamAuditLogRepository auditLogRepository;

public List<AuditLogEntity> buscarAuditoria(UUID userId) {
    return auditLogRepository.findByUserId(userId);
}
```

---

## 8. Limpeza automática de tokens

O módulo executa automaticamente a limpeza de tokens expirados:

- **02:00** — tokens de autenticação expirados
- **02:30** — tokens de confirmação expirados

Nenhuma configuração necessária. Basta que `@EnableScheduling` esteja ativo (o módulo já habilita).

---

## 9. Use Cases disponíveis — Referência rápida

Para injetar qualquer use case, basta declarar como dependência:

```java
@RequiredArgsConstructor
public class SeuUseCase {
    private final IamAuthenticateUseCase authenticateUseCase;
    private final IamValidateTokenUseCase validateTokenUseCase;
    private final IamLogoutUseCase logoutUseCase;
    private final IamLogoutAllDevicesUseCase logoutAllDevicesUseCase;
    private final IamCreateIamUserUseCase createUserUseCase;
    private final IamChangePasswordUseCase changePasswordUseCase;
    private final IamResetPasswordUseCase resetPasswordUseCase;
    private final IamGenerateConfirmationUseCase generateConfirmationUseCase;
    private final IamVerifyConfirmationUseCase verifyConfirmationUseCase;
    private final IamCreateRoleUseCase createRoleUseCase;
    private final IamCreatePermissionUseCase createPermissionUseCase;
    private final IamAssignPermissionToRoleUseCase assignPermissionToRoleUseCase;
    private final IamRevokePermissionFromRoleUseCase revokePermissionFromRoleUseCase;
    private final IamAssignRoleToUserUseCase assignRoleToUserUseCase;
    private final IamRevokeRoleFromUserUseCase revokeRoleFromUserUseCase;
    private final IamRegisterAuditEventUseCase registerAuditUseCase; // eventos customizados de negócio
}
```

---

## 10. Checklist de integração

- [ ] Adicionar dependência `br.com.cati:iam-module:1.0.0` no `pom.xml`
- [ ] Configurar `scanBasePackages` para incluir `br.com.cati.iam`
- [ ] Configurar datasource no `application.yml`
- [ ] Criar `SecurityFilterChain` com o `IamTokenAuthFilter`
- [ ] Criar endpoints de auth (login, logout) chamando os use cases
- [ ] Criar endpoint de criação de usuário (`ROLE_ADMINISTRADOR` apenas) chamando `IamCreateIamUserUseCase`
- [ ] Implementar fluxo de reset de senha (gerar token + enviar email + resetar)
- [ ] Criar as roles e permissions dos módulos do sistema (seed no startup ou migration)
- [ ] Configurar `@ControllerAdvice` para tratar exceptions do IAM
- [ ] Usar `@PreAuthorize` para proteger endpoints por permissão
- [ ] Implementar eventuais resolvers de escopo de dados específicos do domínio e aplicá-los nos use cases relevantes
- [ ] Registrar eventos de auditoria customizados nos fluxos de negócio críticos
- [ ] (Opcional) Sobrescrever configs default no `application.yml`

---

## 11. Troubleshooting

### "Cannot find a matching bean"

O Spring não encontrou os beans do IAM. Verifique se `scanBasePackages` inclui `br.com.cati.iam`.

### Token sempre inválido

1. Verifique se o token não expirou (default: 30 min)
2. Verifique se o `IamTokenAuthFilter` está registrado no `SecurityFilterChain`
3. Verifique se o usuário tem `active = true`

### Conta bloqueada

A conta é bloqueada após 5 tentativas (default). O bloqueio dura 30 min. Para desbloquear manualmente, atualize no banco:

```sql
UPDATE iam_users
SET failed_attempts = 0, account_non_locked = true, lockout_until = NULL
WHERE email = 'usuario@email.com';
```

### RBAC não funciona

1. Verifique se as roles começam com `ROLE_`
2. Verifique se o usuário tem a role atribuída (tabela `iam_user_context_roles`)
3. Verifique se a permission está vinculada à role (tabela `iam_role_permissions`)

### Sessão expirou inesperadamente

O módulo limita o número de sessões ativas por usuário (default: 5). Se o usuário fizer login em mais dispositivos que o limite, as sessões mais antigas são removidas automaticamente. Ajuste com `iam.auth.max-active-sessions` (0 = sem limite).

### Cache desatualizado

Após alterar roles/permissions programaticamente, o cache é limpo automaticamente. Se precisar limpar manualmente:

```java
@Autowired
private IamCacheService cacheService;

cacheService.evictAll();
```
