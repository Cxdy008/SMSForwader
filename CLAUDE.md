# CLAUDE.md

Este ficheiro orienta o Claude Code ao trabalhar neste repositório.

## Visão geral do projeto

**SMSForwarder** é uma aplicação Android nativa (Kotlin) que intercepta SMS recebidas no dispositivo e encaminha-as diretamente para um canal externo (Telegram Bot API, por defeito), sem backend intermediário. Uso pessoal, num único dispositivo, com destino fixo.

A arquitetura completa, o plano de desenvolvimento por fases e a justificação das decisões de design (incluindo por que não há backend) estão documentados em **`/docs/arquitectura.md`** — consultar esse ficheiro antes de propor mudanças estruturais (nova camada, novo canal de saída, reintrodução de backend, etc.), e mantê-lo atualizado sempre que uma decisão de arquitetura mudar.

## Stack técnica

- **Linguagem**: Kotlin
- **UI**: Jetpack Compose
- **Persistência local**: Room (fila/log de mensagens)
- **Background/retry**: WorkManager (`CoroutineWorker`, backoff exponencial)
- **Rede**: Retrofit + OkHttp (chamada direta à API do canal, ex. `api.telegram.org`)
- **Configurações**: DataStore (preferences) + `androidx.security:security-crypto` para o token do bot
- **Build**: Gradle Kotlin DSL (`build.gradle.kts`), version catalog em `libs.versions.toml`

## Identificadores do projeto

- `applicationId` / package raiz: `com.cxdy.smsforwader`
- Manter o package Kotlin e o `applicationId` sincronizados — um rename de package via refactor do Android Studio não atualiza o `applicationId` automaticamente, é preciso editar `build.gradle.kts (Module :app)` à parte.

## Estrutura de pacotes

```
com.cxdy.smsforwader/
├── receiver/   # BroadcastReceiver para SMS_RECEIVED
├── service/    # Foreground Service
├── data/       # Entidades Room, DAO, repositório
├── network/    # Retrofit, clientes de API do canal (Telegram, etc.)
├── worker/     # CoroutineWorker / WorkManager
└── ui/         # Ecrãs Compose
```

## Comandos comuns

```bash
./gradlew assembleDebug          # build de debug
./gradlew assembleRelease        # build de release (sideload, sem Play Store)
./gradlew test                   # testes unitários
./gradlew connectedAndroidTest   # testes instrumentados (requer dispositivo/emulador)
adb install app/build/outputs/apk/release/app-release.apk
```

## Convenções e decisões a respeitar

- **Sem backend**: o app fala diretamente com a API do canal de saída. Não introduzir um servidor intermediário sem atualizar `/docs/arquitectura.md` primeiro e justificar a mudança (ex: necessidade real de múltiplos canais, múltiplos dispositivos, ou roteamento condicional — ver secção "Caminho de evolução" nesse documento).
- **Token do bot nunca em texto plano**: usar `security-crypto` para guardar credenciais em disco; nunca commitar tokens no repositório.
- **Registo do `BroadcastReceiver`**: dinâmico (`registerReceiver` em runtime), não estático no manifest, para compatibilidade com versões recentes do Android.
- **Distribuição**: sideload direto do APK assinado — não há submissão à Play Store (política restritiva para permissões `RECEIVE_SMS`/`READ_SMS`).
- **Gestão de bateria**: o app depende de isenção de otimização de bateria (`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`); em dispositivos Xiaomi/Huawei/Samsung pode ser necessária configuração manual adicional fora do app.

## Ao propor mudanças

Antes de alterar fluxo de dados, adicionar um novo canal de encaminhamento, ou mexer na camada de retry/fila, ler `/docs/arquitectura.md` para confirmar que a mudança é consistente com as decisões já tomadas — e atualizar esse documento como parte da mesma alteração, não como passo separado.