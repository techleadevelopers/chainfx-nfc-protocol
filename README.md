# ChainFX NFC Closed-Loop Rail

Este pacote implementa a parte backend do pagamento NFC fechado da ChainFX. Ele nao cria, por si so, o servico Android HCE. O HCE fica no app mobile Android, usando `HostApduService`, e deve enviar ao terminal um token opaco emitido por este backend.

## O que foi implementado

- Token dinamico `nfc1.<payload>.<signature>` assinado por HMAC-SHA256.
- Claims minimas: `token_id`, `wallet`, `device_id`, `network`, `iat`, `exp`, `nonce`.
- Hash do token persistido, nunca dependencia de PAN/Track2.
- Autorizacao transacional com idempotencia.
- Ledger NFC simples com `available_usdt_micro` e `locked_usdt_micro`.
- Hold de USDT ao aprovar uma transacao.
- Respostas no estilo autorizador:
  - `00`: aprovado.
  - `51`: saldo insuficiente.
  - `05`: recusado.

## O que nao foi implementado ainda

- O app Android HCE em si dentro deste repositorio.
- A adaptacao do `CardApduService.java` do projeto `nfcemv-emulator`.
- Integracao com POS Visa/Mastercard/adquirente.
- Certificacao EMVCo, PCI DSS, BIN sponsor ou issuer processor.
- HCE iOS em producao. No iOS, HCE de pagamento depende de recursos/entitlements e regras da Apple.

Este trilho e closed-loop: funciona com app ChainFX + leitor/terminal ChainFX. Um POS comum de adquirente nao vai rotear automaticamente para `/api/nfc/authorize`.

## Fluxo tecnico

1. O app autenticado chama `POST /api/nfc/provision`.
2. O backend emite token HMAC com TTL curto, por padrao 120 segundos.
3. O app Android HCE responde ao APDU do terminal com esse token opaco.
4. O terminal ChainFX envia o token para `POST /api/nfc/authorize`.
5. O backend verifica assinatura, expiracao, token persistido e idempotencia.
6. O backend calcula o USDT necessario usando a cotacao `USDT/BRL`.
7. O banco trava a linha de saldo com `SELECT ... FOR UPDATE`.
8. Se `available_usdt_micro >= required_usdt_micro`, move saldo para `locked_usdt_micro` e aprova.
9. Se nao houver saldo, grava a autorizacao como `requires_funding`.

## Endpoints

### Provisionamento

```http
POST /api/nfc/provision
Authorization: Bearer sk_test_...
Content-Type: application/json
```

```json
{
  "wallet_address": "0x742d35cc6634c0532925a3b844bc454e4438f44e",
  "device_id": "android-device-id",
  "network": "BSC",
  "ttl_seconds": 120
}
```

Resposta:

```json
{
  "token": "nfc1...",
  "token_id": "...",
  "expires_at": "2026-07-18T13:53:00Z",
  "network": "BSC"
}
```

### Autorizacao

```http
POST /api/nfc/authorize
Authorization: Bearer sk_test_...
Idempotency-Key: terminal-tx-001
Content-Type: application/json
```

```json
{
  "token": "nfc1...",
  "amount_brl": "25.90",
  "currency": "BRL",
  "merchant_id": "merchant_demo",
  "terminal_id": "terminal_01",
  "external_ref": "cupom-123",
  "idempotency_key": "terminal-tx-001"
}
```

Resposta aprovada:

```json
{
  "authorization_id": "nfc_auth_...",
  "status": "approved",
  "response_code": "00",
  "required_usdt": "4.712345",
  "hold_expires_at": "2026-07-18T14:08:00Z"
}
```

Resposta sem saldo:

```json
{
  "status": "requires_funding",
  "response_code": "51",
  "reason": "insufficient_usdt"
}
```

### Funding sandbox

Disponivel apenas com `ALLOW_SIMULATIONS=true`.

```http
POST /api/nfc/sandbox/fund
```

```json
{
  "wallet_address": "0x742d35cc6634c0532925a3b844bc454e4438f44e",
  "network": "BSC",
  "amount_usdt": "100.000000"
}
```

Em producao, esse saldo deve vir de deposito/escrow on-chain reconciliado pelo backend.

## Schema

Migration principal:

```text
migrations/020_nfc_closed_loop.sql
```

Tabelas:

- `nfc_tokens`: token opaco, hash, wallet, device, rede, expiracao e status.
- `nfc_wallet_balances`: saldo disponivel/travado por wallet, rede e asset.
- `nfc_authorizations`: autorizacoes idempotentes, merchant, terminal, valor, taxa, status e hold.

## Variaveis

```env
NFC_ENABLED=true
NFC_TOKEN_SECRET=use-um-segredo-forte
NFC_TOKEN_TTL_SEC=120
NFC_HOLD_TTL_SEC=900
NFC_MAX_AMOUNT_BRL=500
```

Em producao, `NFC_TOKEN_SECRET` e obrigatorio quando `NFC_ENABLED=true`.

## Integracao Android HCE

O projeto `nfcemv-emulator` ja tem os pontos certos:

- `CardApduService.java`: recebe APDU.
- `apduservice.xml`: registra AIDs.
- `ApduUtils.java`: helpers de HEX/APDU.

O ajuste correto e substituir o Track2/PAN estatico por token opaco:

1. Antes da aproximacao, app chama `/api/nfc/provision`.
2. App guarda o token somente em memoria.
3. `CardApduService.processCommandApdu` responde ao leitor com esse token em um registro proprietario.
4. Leitor ChainFX extrai o token e chama `/api/nfc/authorize`.

Nao usar PAN real, CVV, Track2 real ou dados de bandeira nesse fluxo.

## Latencia medida

Ambiente:

- Windows, PowerShell.
- Pacote: `payment-gateway/internal/nfc`.
- Teste: `TestTokenLatencyPercentiles`.
- Operacao medida: `IssueToken + VerifyToken`.
- Amostras: 1000 lotes.
- Tamanho do lote: 100 operacoes.
- Total: 100000 operacoes.

Comando:

```powershell
go test ./internal/nfc -run TestTokenLatencyPercentiles -count=1 -v
```

Resultado desta maquina:

```text
p50 = 9.973us
p55 = 9.987us
p95 = 100.645us
p99 = 101.557us
max = 116.765us
```

Leitura:

- O custo criptografico local do token nao e gargalo.
- A latencia real de autorizacao NFC sera dominada por HTTP, Postgres, cotacao `USDT/BRL` e rede do terminal.
- Para producao, medir tambem `/api/nfc/authorize` end-to-end com Postgres real e carga concorrente.

## Testes

```powershell
go test ./internal/nfc
go test ./internal/nfc ./internal/database ./internal/server
go build -o api-nfc.exe ./cmd/api
```

Ultima validacao local:

```text
go test ./internal/nfc ./internal/database ./internal/server
ok

go build -o api-nfc.exe ./cmd/api
ok

GET http://127.0.0.1:18080/healthz
200 {"ok":true}
```
