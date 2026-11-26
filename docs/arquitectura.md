# SMSForwarder — Documento de Arquitetura e Plano de Desenvolvimento

## 1. Visão Geral

O SMSForwarder é uma aplicação Android nativa (Kotlin) responsável por interceptar mensagens SMS recebidas num dispositivo e encaminhá-las diretamente para um canal externo (Telegram, email, etc.), sem backend intermediário — arquitetura adequada para uso pessoal, num único dispositivo, com destino fixo.

**Objetivo principal:** garantir que nenhuma mensagem se perde, mesmo em condições adversas de conectividade, gestão agressiva de bateria dos fabricantes, ou falhas temporárias do canal de destino.

---

## 2. Arquitetura Geral

```
┌─────────────────────────────────────────────────────────────┐
│                    DISPOSITIVO ANDROID                       │
│                                                                │
│  ┌──────────────┐      ┌───────────────┐     ┌────────────┐  │
│  │ BroadcastRcvr│─────▶│ Foreground     │────▶│ WorkManager│  │
│  │ (SMS_RECEIVED)│      │ Service        │     │ (fila/retry)│ │
│  └──────────────┘      └───────────────┘     └─────┬──────┘  │
│                                ▲                     │         │
│                          ┌─────┴─────┐         ┌─────▼──────┐ │
│                          │ Room DB   │         │ Retrofit/   │ │
│                          │ (log/fila)│         │ OkHttp      │ │
│                          └───────────┘         └─────┬──────┘ │
└──────────────────────────────────────────────────────┼────────┘
                                                          │ HTTPS
                                                          ▼
                                            ┌──────────────────────┐
                                            │   Telegram Bot API    │
                                            │   (ou outro canal)    │
                                            └──────────────────────┘
```

Sem servidor próprio no meio: o app fala diretamente com a API do canal escolhido (ex: `api.telegram.org`). Isto elimina uma camada inteira de infraestrutura a manter, à custa de menos flexibilidade de roteamento — um compromisso aceitável quando só tu usas a app e o destino é fixo.

### 2.1 Camadas

| Camada | Responsabilidade |
|---|---|
| **Interceção** | `BroadcastReceiver` captura o SMS assim que chega ao sistema |
| **Persistência local** | `Room` guarda cada mensagem e o estado do encaminhamento (pendente, enviado, falhado) |
| **Execução em segundo plano** | `Foreground Service` mantém o processo vivo; `WorkManager` garante retry mesmo com a app fechada |
| **Transporte** | `Retrofit`/`OkHttp` envia o payload diretamente à API do canal escolhido (ex: Telegram) via HTTPS |
| **Canal de saída** | Telegram Bot API (recomendado pela simplicidade), ou alternativa (SMTP, etc.) |

### 2.2 Nota sobre segurança do token

Sem backend, o token do bot (ou credenciais do canal escolhido) fica embutido no APK — seja em código, `BuildConfig` ou `DataStore` encriptado. Como o app é só teu e não vai ser distribuído, o risco é baixo, mas vale a pena:
- Nunca commitar o token em texto plano no repositório (usar `local.properties` + `BuildConfig.buildConfig` ou variável de ambiente no build)
- Usar `androidx.security:security-crypto` para guardar o token encriptado em disco em vez de `SharedPreferences`/`DataStore` simples
- Se o bot Telegram for dedicado só a isto, o impacto de um eventual vazamento do token é limitado (dá só para enviar mensagens ao teu chat, não acesso à tua conta)

---

## 3. Plano de Desenvolvimento — Passo a Passo

### Fase 1 — Fundação do projeto Android
1. Criar projeto Kotlin + Jetpack Compose no Android Studio
2. Configurar `build.gradle.kts` com as dependências base
3. Definir estrutura de pacotes (`receiver`, `service`, `data`, `network`, `ui`, `worker`)
4. Configurar `AndroidManifest.xml` com permissões: `RECEIVE_SMS`, `READ_SMS`, `INTERNET`, `FOREGROUND_SERVICE`, `POST_NOTIFICATIONS` (Android 13+)

### Fase 2 — Interceção de SMS
5. Implementar `BroadcastReceiver` para `SMS_RECEIVED_ACTION`
6. Extrair remetente, corpo da mensagem e timestamp do `Bundle` (`pdus`)
7. Registar o receiver dinamicamente (`registerReceiver` em runtime) em vez de estático, para compatibilidade com versões recentes
8. Testar receção em emulador via `adb emu sms send`

### Fase 3 — Persistência local
9. Modelar entidade `SmsMessage` (id, remetente, corpo, timestamp, estado, tentativas)
10. Criar `Room` database com DAO para inserir/atualizar/consultar mensagens
11. Implementar repositório (`SmsRepository`) como camada de abstração entre UI/Worker e a base de dados

### Fase 4 — Execução em segundo plano
12. Implementar `Foreground Service` com notificação persistente de baixo nível de prioridade
13. Pedir isenção de otimização de bateria (`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`)
14. Adicionar orientação na UI para desativar a gestão agressiva de bateria, caso o dispositivo seja de um fabricante como Xiaomi/Huawei/Samsung

### Fase 5 — Encaminhamento e fila
15. Implementar `WorkManager` com `CoroutineWorker` para processar a fila de mensagens pendentes
16. Configurar política de retry com backoff exponencial
17. Implementar chamada HTTP via `Retrofit` diretamente para a API do canal escolhido (ex: `https://api.telegram.org/bot<TOKEN>/sendMessage`)
18. Atualizar estado da mensagem no `Room` conforme resposta (sucesso/falha)

### Fase 6 — UI da aplicação
19. Ecrã principal: estado do serviço (ativo/inativo), histórico de mensagens encaminhadas
20. Ecrã de configuração: token do bot, chat ID de destino, formato da mensagem
21. Onboarding: pedido de permissões passo a passo, explicação da necessidade de isenção de bateria

### Fase 7 — Testes e robustez
22. Testes unitários do repositório e da lógica de formatação/envio
23. Testes de integração: simular perda de rede, reinício do dispositivo, app morta pelo sistema
24. Teste em condições reais no teu próprio dispositivo por alguns dias antes de confiar na app para uso contínuo

### Fase 8 — Distribuição
25. Build do APK de release (sideload — não há necessidade de Play Store para uso pessoal)
26. Assinar o APK com keystore próprio
27. Instalar diretamente no dispositivo (`adb install` ou transferência manual)

---

## 4. Bibliotecas Necessárias

| Biblioteca | Finalidade |
|---|---|
| `androidx.core:core-ktx` | Extensões Kotlin para APIs do Android |
| `androidx.compose.*` (BOM) | UI declarativa (Jetpack Compose) |
| `androidx.room:room-runtime` + `room-ktx` + `room-compiler` (KSP) | Persistência local (fila/log de mensagens) |
| `androidx.work:work-runtime-ktx` | Agendamento e retry de encaminhamento em segundo plano |
| `androidx.lifecycle:lifecycle-service` | Suporte a `Foreground Service` com lifecycle |
| `com.squareup.retrofit2:retrofit` | Cliente HTTP para comunicação com a API do canal (Telegram, etc.) |
| `com.squareup.retrofit2:converter-gson` (ou `converter-moshi`) | Serialização JSON |
| `com.squareup.okhttp3:okhttp` + `logging-interceptor` | Camada de transporte HTTP, debug de requisições |
| `androidx.datastore:datastore-preferences` | Guardar configurações (chat ID, formato de mensagem) |
| `androidx.security:security-crypto` | Encriptação do token do bot em disco |
| `kotlinx.coroutines` | Programação assíncrona (Workers, chamadas de rede) |
| `androidx.navigation:navigation-compose` | Navegação entre ecrãs |

Não é necessária nenhuma biblioteca de backend — toda a stack é Android/Kotlin.

---

## 5. Pontos de Atenção Críticos

- **Restrições de fabricante**: Xiaomi (MIUI), Huawei (EMUI), Samsung e outros têm camadas próprias de gestão de bateria que matam serviços em segundo plano mesmo com `Foreground Service` ativo. É necessário desativar essas otimizações manualmente no teu próprio dispositivo.
- **Sem necessidade de Play Store**: como é uso pessoal, a distribuição é por sideload direto do APK — evita completamente as restrições da Play Store sobre `RECEIVE_SMS`/`READ_SMS`.
- **Privacidade e segurança**: o conteúdo dos SMS é dado sensível — a comunicação com a API do canal deve ser sempre via HTTPS, e o token do bot deve ficar encriptado em disco, nunca em texto plano no código-fonte.
- **Backup desativado**: `android:allowBackup="false"` e todas as regras de `backup_rules.xml` (API 29–30) / `data_extraction_rules.xml` (API 31+) excluem tudo, incluindo a transferência dispositivo-a-dispositivo (que no Android 12+ não é desligada só pelo `allowBackup`). Motivos: o conteúdo dos SMS no Room não deve sair do dispositivo, e o token encriptado não é recuperável após um restauro (a chave do Android Keystore não é incluída no backup). Consequência: ao mudar de telemóvel, a configuração (token, chat ID) tem de ser feita de novo.
- **Confiabilidade de entrega**: a combinação `Room` (estado local) + `WorkManager` (retry com backoff) é suficiente, sem backend, para garantir que uma mensagem não se perde em falhas temporárias de rede — desde que o canal de destino (Telegram, por exemplo) tenha boa disponibilidade.
- **Caminho de evolução**: se no futuro precisares de múltiplos canais, filtros de roteamento, ou vários dispositivos a reportar ao mesmo sítio, o backend intermediário (FastAPI/Spring Boot) volta a fazer sentido — a arquitetura atual não impede essa evolução, só a adia até ser realmente necessária.