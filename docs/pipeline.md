# Pipeline de Encaminhamento — SMSForwarder

> **Estado:** desenho. Nenhuma etapa está implementada ainda; os caminhos de ficheiro abaixo são os planeados e devem ser corrigidos aqui quando o código existir (ver [roadmap](roadmap.md), Fases 2–5).

Este documento descreve o caminho de uma SMS desde que chega ao dispositivo até ser entregue no Telegram, e as regras que esse caminho nunca pode quebrar. A justificação das escolhas está em [arquitectura.md](arquitectura.md); aqui fica o *como*.

## 1. Etapas

```
SMS recebida pelo sistema
        │  broadcast SMS_RECEIVED (protegido por BROADCAST_SMS)
        ▼
SmsReceiver                        receiver/SmsReceiver.kt          (planeado)
        │  getMessagesFromIntent() → RawSms
        ▼
Regras puras (sem Android)         receiver/SmsCapture.kt           (planeado)
        │  junta multipartes, valida → SmsMessage
        ▼
Room: insere com estado PENDING    data/SmsRepository.kt            (planeado)
        │  enqueueUniqueWork("send-<id>", KEEP)
        ▼
ForwardSmsWorker                   worker/ForwardSmsWorker.kt       (planeado)
        │  formata + divide em blocos ≤ 4096 caracteres
        ▼
Telegram Bot API (sendMessage)     network/TelegramClient.kt        (planeado)
        │  200 OK → estado SENT, sentAt = agora
        ▼
Purga do corpo (≥ 1 min depois)    worker/PurgeBodyWorker.kt        (planeado)
```

- **Receiver**: corre dentro de `goAsync()` (janela de ~10 s). Só converte o intent em `RawSms`, grava e enfileira. Nunca faz rede.
- **`RawSms` e regras puras**: o receiver converte os PDUs numa estrutura simples, independente do Android (remetente, corpo, timestamp, subId). A lógica de juntar multipartes, formatar e dividir mensagens vive em classes Kotlin puras, testáveis na JVM sem emulador.
- **Trabalho único por mensagem**: o nome do trabalho deriva do id do Room, com política `KEEP` — enfileirar duas vezes a mesma SMS não gera dois envios.
- **Worker**: trabalho *expedited* com restrição de rede e `OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST` (sem quota expedited, corre como trabalho normal em vez de falhar); em API < 31 corre como foreground e precisa de `getForegroundInfo()`. Retry com backoff exponencial.

## 2. Contrato de segurança

Estas regras são invariantes. Uma alteração que quebre alguma delas tem de ser bloqueada em revisão, mesmo que o resto do código esteja correto.

1. **Gravar antes de tudo**: a SMS é gravada no Room antes de qualquer outra operação. Se a gravação falhar, nada é enfileirado e o erro é registado (sem conteúdo).
2. **O receiver nunca faz rede**, nem trabalho que possa exceder a janela do `goAsync()`.
3. **Uma mensagem só passa a `SENT` depois de um 200 OK do Telegram** para todos os seus blocos. Qualquer outra resposta, timeout ou exceção deixa-a `PENDING` e o WorkManager volta a tentar.
4. **Mensagens `PENDING` nunca são apagadas** — nem pela purga, nem por limpeza de histórico, nem por limite de tentativas.
5. **O corpo só é apagado depois do envio confirmado** (secção 4); nunca antes.
6. **Logs**: em debug, só ids, estados, número de tentativas e códigos HTTP — nunca o remetente, o corpo, o token ou o chat ID. Em release, sem logs de mensagens.
7. **Erros permanentes** (ex.: 401 token inválido, 400 chat inexistente) não são tratados como sucesso: a mensagem fica `PENDING` e a UI mostra o problema de configuração. Corrigida a configuração, os envios retomam.

## 3. Estados

```
PENDING ──200 OK (todos os blocos)──▶ SENT ──≥ 1 min──▶ SENT (corpo apagado)
   ▲  │
   └──┘ falha / timeout / sem rede → retry com backoff
```

| Estado | Significado |
|---|---|
| `PENDING` | Gravada, ainda não confirmada pelo Telegram. Pode estar a aguardar rede ou retry. |
| `SENT` | Todos os blocos confirmados com 200 OK. `sentAt` registado. |

Não existe estado `FAILED` terminal: uma mensagem nunca desiste de ser enviada (objetivo principal). Falhas repetidas são visíveis na UI pelo número de tentativas e pelo último erro.

**Duplicados**: se o Telegram aceitar a mensagem mas a resposta se perder, o retry envia-a de novo. É aceite — uma mensagem duplicada é preferível a uma perdida.

## 4. Retenção do conteúdo

O texto da SMS é dado sensível e não deve ficar no dispositivo mais tempo do que o necessário.

- **1 minuto depois do 200 OK, o corpo da mensagem é apagado** do Room (o campo passa a vazio/`null`). O registo mantém-se com os metadados — id, remetente, timestamp de receção, `sentAt`, tentativas — para o histórico na UI.
- Mecanismo: ao marcar `SENT`, o worker enfileira um `PurgeBodyWorker` único por mensagem com `setInitialDelay(1 min)`. Como o WorkManager pode atrasar trabalho sob Doze, há também uma varredura de segurança: cada execução do worker (e o arranque da app) apaga o corpo de todas as mensagens `SENT` com `sentAt` há mais de 1 minuto. O minuto é, portanto, um mínimo; o máximo depende de quando o sistema deixa a app correr.
- Mensagens `PENDING` mantêm sempre o corpo — sem ele não podem ser enviadas.

## 5. Formatação e limites

- SMS multipartes chegam como vários PDUs no mesmo broadcast: concatenar os corpos do mesmo remetente pela ordem recebida.
- O Telegram aceita no máximo 4096 caracteres por mensagem: dividir em blocos, sem cortar a meio de um carácter (pares surrogate / emoji). Cada bloco é um `sendMessage`; a mensagem só fica `SENT` com todos confirmados.
- Formato e escape dependem do `parse_mode` escolhido (decisão da Fase 5); o corpo da SMS é texto do utilizador e tem de ser escapado ou enviado sem `parse_mode`.

## 6. Limitações conhecidas

Ver [arquitectura.md](arquitectura.md), secção 5: SMS antes do primeiro desbloqueio, app forçada a parar e restrições de arranque automático dos fabricantes.
