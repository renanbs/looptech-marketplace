# Índice defensivo — Anthropic Cybersecurity Skills

Fonte opcional: [mukul975/Anthropic-Cybersecurity-Skills](https://github.com/mukul975/Anthropic-Cybersecurity-Skills)
(Apache-2.0, community, **não** afiliado à Anthropic PBC). ~800 skills; a
maioria é dual-use ou ofensiva. Este workflow **não** vendora a
biblioteca e **não** carrega skill de exploit, C2, phishing, red team
ou pentest ativo.

Usamos só o **método** (When to Use → Prerequisites → Workflow de
*checagem* → Verification) e o mapeamento de frameworks (ATT&CK como
rótulo de ameaça, D3FEND / NIST CSF como remédio). A prova continua
sendo o diff + scanners do repo + teste que o `impl` deve adicionar.

## Como carregar

1. Se o humano tiver a biblioteca instalada (marketplace / clone
   local), o orquestrador cola no handoff **no máximo 3** skills da
   allowlist abaixo cujo domínio casa com a superfície do diff.
2. Se a biblioteca **não** estiver instalada, **não** clone, **não**
   rode `npx skills add`. Use esta página + `security-review.md` +
   `pentest.md`. O review não fica bloqueado.
3. Do corpo de uma skill allowlisted, leia só: intenção, o que
   procurar no código, verificação. Ignore scripts/, wordlists,
   payloads e qualquer passo que execute a falha.

## Allowlist (defensivo / modelagem / detecção em fonte)

| Superfície no diff | Skill (nome upstream) | O que extrair |
|---|---|---|
| Qualquer mudança de auth, API, pagamento | `implementing-threat-modeling-with-mitre-attack` | ator → ação → prejuízo; sem emulação |
| Rota / gateway / log de API | `analyzing-api-gateway-access-logs` | o que o handler não deve vazar em log |
| Dependência / lockfile | `hunting-for-supply-chain-compromise` | sinais no lock; não “quebra o pacote” |
| Upload, HTML, template | `hunting-for-webshell-activity` | sinks (`innerHTML`, eval, path de upload) |
| Role, tenant, admin | `detecting-privilege-escalation-attempts` | papel vindo do client; IDOR |
| Segredo, token, cookie | `detecting-credential-dumping-techniques` | segredo no git/log/client — só presença |
| Coverage / lacuna de controle | `implementing-mitre-attack-coverage-mapping` | lacuna → remédio no código |

Três no máximo por review. O resto da biblioteca não entra no
contexto.

## Denylist — NÃO carregar, NÃO resumir, NÃO seguir

Qualquer skill cujo nome comece com ou contenha:

- `abusing-`, `exploiting-`, `weaponiz`
- `conducting-full-scope-red-team`, `conducting-spearphishing`,
  `conducting-social-engineering`, `conducting-pass-the-ticket`,
  `conducting-domain-persistence`, `conducting-internal-network-penetration`,
  `conducting-cloud-penetration`
- `building-red-team-c2`, `performing-initial-access`,
  `performing-credential-access-with`, `performing-privilege-escalation-on`
- `executing-red-team`, `performing-purple-team-atomic-testing` (emulação)

Se o handoff pedir uma dessas: recuse em uma frase e volte à tabela
de `security-review.md`.

## Mapa das nossas classes → frameworks (sem TTP operacional)

| Classe (security-review.md) | ATT&CK (rótulo) | CSF / D3FEND (intenção) |
|---|---|---|
| Segredo | T1552 | PR.DS — não commitar, não logar |
| Authz / IDOR | T1078 | PR.AA — ownership no servidor |
| Authn | T1078 / T1539 | PR.AA — cookie/token não forjável |
| Injection | T1190 | D3-IAA — parâmetro ligado, teste de regressão |
| Dinheiro / corrida | T1496 (impacto) | PR.IP — lock na mesma transação |
| PII | TA0010 | PR.DS / LGPD — redigir na borda |
| Superfície interna | T1190 | PR.IR — gate de ambiente |
| SSRF / upload / redirect | T1190 | D3-UAN — allowlist, tipo/tamanho |
| Crypto / token | T1552 / T1539 | PR.DS — secret de env, expiração |

Os IDs são **classificação** no relatório, não um convite a reproduzir
a técnica.

## Relatório

Em Evidências, se usou a biblioteca:

```
Cyber skills: <nome1>, <nome2> (allowlist; instalada)
```

Se não:

```
Cyber skills: índice local only (biblioteca ausente)
```

Nunca cole workflow ofensivo no retorno.
