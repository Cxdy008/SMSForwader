# Guia de Desenvolvimento — SMSForwarder

As versões não estão repetidas aqui de propósito. A fonte de verdade é:

| O quê | Onde |
|---|---|
| AGP, Kotlin, bibliotecas | `gradle/libs.versions.toml` |
| `compileSdk`, `targetSdk`, `minSdk` | `app/build.gradle.kts` |
| JDK do daemon Gradle | `gradle/gradle-daemon-jvm.properties` |
| Versão do Gradle | `gradle/wrapper/gradle-wrapper.properties` |

## Requisitos

- Android Studio recente, ou só a linha de comandos com o Android SDK (`sdk.dir` em `local.properties`, que não é commitado)
- O JDK indicado em `gradle-daemon-jvm.properties` — o Gradle descarrega-o se não existir
- A plataforma do `compileSdk` é instalada automaticamente no primeiro build, se as licenças do SDK estiverem aceites
- Um emulador ou dispositivo com Android ≥ `minSdk`; para testar SMS de verdade, um dispositivo com SIM

## Comandos

```bash
./gradlew assembleDebug          # APK em app/build/outputs/apk/debug/
./gradlew test                   # testes unitários (JVM)
./gradlew lint                   # relatório em app/build/reports/
./gradlew connectedAndroidTest   # testes instrumentados (emulador/dispositivo)
./gradlew assembleRelease        # ver "Build de release assinado"
```

Antes de abrir um PR: `./gradlew test lint assembleDebug` tem de passar.

## Testar SMS no emulador

```bash
adb emu sms send 5551234 "Olá do emulador"          # SMS simples
adb emu sms send 5551234 "$(printf 'x%.0s' {1..400})" # SMS longa (multipartes)
```

Outros comandos úteis para os testes manuais:

```bash
adb shell am kill com.cxdy.smsforwarder               # termina o processo (como o sistema faria)
adb shell am force-stop com.cxdy.smsforwarder         # "Forçar paragem": deixa de receber broadcasts
adb shell cmd connectivity airplane-mode enable       # sem rede (disable para voltar)
adb shell dumpsys deviceidle force-idle               # força o Doze (unforce para sair)
adb shell pm revoke com.cxdy.smsforwarder android.permission.RECEIVE_SMS
adb shell dumpsys jobscheduler | grep -A3 smsforwarder # trabalho agendado pelo WorkManager
adb reboot
```

## Testes manuais

A Fase 7 do [roadmap](roadmap.md) só fica concluída quando todos passam num dispositivo real. Cada teste parte da app configurada (token e chat ID válidos) e com `RECEIVE_SMS` concedida.

| # | Cenário | Passos | Resultado esperado |
|---|---|---|---|
| 1 | Envio normal | App aberta; enviar SMS | Chega ao Telegram; estado `SENT` no histórico |
| 2 | App terminada | `am kill`; enviar SMS | Chega ao Telegram sem abrir a app |
| 3 | SMS longa | Enviar SMS de 400+ caracteres | Chega como uma só mensagem, texto completo |
| 4 | Sem rede | Modo de avião; enviar SMS; esperar; desligar modo de avião | Fica `PENDING`; é enviada quando a rede volta |
| 5 | Reinício com pendentes | Teste 4 até `PENDING`; `adb reboot`; desbloquear; ligar a rede | É enviada após o reinício |
| 6 | Doze | `force-idle`; enviar SMS | É enviada (eventualmente adiada); nunca perdida |
| 7 | Token inválido | Configurar token errado; enviar SMS; corrigir token | Fica `PENDING` com erro visível; enviada após a correção |
| 8 | Purga do corpo | Teste 1; esperar > 1 min | O histórico mantém a entrada, sem o texto da SMS |
| 9 | Permissão revogada | `pm revoke RECEIVE_SMS`; abrir a app | Ecrã principal mostra que não está a funcionar |
| 10 | Forçar paragem | `force-stop`; enviar SMS | Não é encaminhada (limitação aceite); abrir a app repõe o funcionamento |
| 11 | Sem fugas nos logs | `adb logcat` durante os testes 1–7 | Nenhum remetente, texto, token ou chat ID aparece |

## Build de release assinado

O keystore e as passwords **nunca** entram no repositório (o `.gitignore` exclui `*.jks` e `*.keystore`).

1. Criar o keystore uma vez, fora do projeto, e guardar uma cópia de segurança — sem ele não é possível instalar atualizações por cima da versão já instalada:

   ```bash
   keytool -genkeypair -v -keystore ~/.android-keystores/smsforwarder.jks \
     -alias smsforwarder -keyalg RSA -keysize 4096 -validity 10000
   ```

2. Pôr as credenciais em `~/.gradle/gradle.properties` (global, fora do repositório):

   ```properties
   SMSF_KEYSTORE=/home/<utilizador>/.android-keystores/smsforwarder.jks
   SMSF_KEYSTORE_PASSWORD=...
   SMSF_KEY_ALIAS=smsforwarder
   SMSF_KEY_PASSWORD=...
   ```

3. O `signingConfigs.release` em `app/build.gradle.kts` lê estas propriedades (a configurar na Fase 8).
4. `./gradlew assembleRelease` e `adb install -r app/build/outputs/apk/release/app-release.apk`.

## Convenções

- **Branches**: `<tipo>/<n.º>-<descrição-curta>`, ex. `docs/4-docs-structure`, `feat/5-sms-receiver`. Tipos: `feat`, `fix`, `docs`, `chore`, `refactor`, `build`.
- **Commits**: [Conventional Commits](https://www.conventionalcommits.org/) em inglês, assunto no imperativo (`feat: add static SMS receiver`). Um assunto por commit; refactors separados de mudanças de comportamento.
- **PRs**: um assunto por PR, template preenchido, docs atualizadas no mesmo PR.
- **Língua**: documentação em `docs/` e `CLAUDE.md` em português; código, commits, PRs e templates do GitHub em inglês.
- **Dependências**: cada biblioteca nova é justificada no PR e, se for estrutural, em [arquitectura.md](arquitectura.md) §4. Sem frameworks de DI (Hilt/Koin) — injeção por construtor.
- **Strings**: texto visível ao utilizador sempre em `res/values/strings.xml`.

## Problemas frequentes

- **`The project is using an incompatible version (AGP …) of the Android Gradle plugin`** (sync falha no Android Studio): cada versão do Android Studio só suporta o AGP até à sua própria linha. Antes de subir o AGP no `libs.versions.toml`, confirmar que o Android Studio instalado o suporta (tabela de compatibilidade em developer.android.com/build/releases/gradle-plugin); caso contrário, atualizar primeiro o Android Studio. Ao subir o AGP, remover também `android.suppressUnsupportedCompileSdk` do `gradle.properties` se já não for necessário.
- **`Minimum supported Gradle version is …`**: o AGP foi atualizado sem o wrapper. Atualizar com `./gradlew wrapper --gradle-version <versão>` (pode ser preciso editar primeiro o `distributionUrl`).
- **`Dependency … requires … compileSdk …`**: uma biblioteca exige um `compileSdk` mais recente. Subir só o `compileSdk`; o `targetSdk` muda o comportamento em runtime e é uma alteração separada.
- **SMS do emulador não chega à app**: confirmar que `RECEIVE_SMS` está concedida e que a app não foi forçada a parar (abrir a app uma vez repõe).
- **Nada é enviado num Xiaomi/Huawei/Samsung**: ver [permissoes.md](permissoes.md) §1 — arranque automático e bateria do fabricante.
