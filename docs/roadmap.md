# Roadmap — SMSForwarder

Fases por ordem de execução; não há datas. Cada fase indica o estado atual — **atualizar o estado no mesmo PR que o altera**.

Legenda: ✅ feito · 🚧 em curso · ⬜ por fazer

## Fase 0 — Repositório e documentação ✅

- Projeto Kotlin + Jetpack Compose criado; `applicationId` `com.cxdy.smsforwarder`
- Toolchain atualizada para AGP 9 com Kotlin integrado
- Backup e transferência entre dispositivos desativados
- Arquitetura e decisões em [arquitectura.md](arquitectura.md); pipeline, permissões e guia de desenvolvimento documentados
- Templates do GitHub (PR, feature, documentação, bug)

## Fase 1 — Fundação do projeto Android ✅

1. ✅ Criar projeto Kotlin + Jetpack Compose
2. ✅ Adicionar ao version catalog as dependências das Fases 2–5 ([arquitectura.md](arquitectura.md) §4)
3. ✅ Criar a estrutura de pacotes (`receiver`, `data`, `network`, `security`, `worker`, `ui`); os pacotes vazios têm um `.gitkeep`, a remover quando receberem o primeiro ficheiro
4. ✅ Declarar no manifest as permissões de [permissoes.md](permissoes.md)

## Fase 2 — Interceção de SMS ⬜

5. Receiver estático para `SMS_RECEIVED_ACTION`, protegido por `BROADCAST_SMS` ([pipeline.md](pipeline.md) §1)
6. Converter o intent em `RawSms` com `getMessagesFromIntent()`; regras puras para juntar multipartes, com testes JVM
7. `goAsync()` para gravar no Room e enfileirar o envio
8. Testes manuais 1–3 de [desenvolvimento.md](desenvolvimento.md#testes-manuais)

## Fase 3 — Persistência local ⬜

9. Entidade `SmsMessage` (id, remetente, corpo, timestamp, estado, tentativas, último erro, `sentAt`)
10. Base de dados Room e DAO
11. `SmsRepository` entre UI/worker e a base de dados

## Fase 4 — Resistência à gestão de bateria ⬜

12. Pedir isenção de otimização de bateria
13. Orientação na UI para o arranque automático e a gestão de bateria dos fabricantes

## Fase 5 — Encaminhamento e fila ⬜

14. `ForwardSmsWorker` expedited (com `getForegroundInfo()` para API < 31)
15. Trabalho único por mensagem, restrição de rede, backoff exponencial
16. Cliente Retrofit da Bot API com `kotlinx.serialization`; token com Keystore ([arquitectura.md](arquitectura.md) §2.3)
17. Formatação, escape e divisão em blocos de 4096 caracteres ([pipeline.md](pipeline.md) §5), com testes JVM
18. Estados `PENDING`/`SENT` e purga do corpo 1 minuto após o envio ([pipeline.md](pipeline.md) §3–4)

## Fase 6 — UI ⬜

19. Ecrã principal: estado das permissões e da configuração, histórico (só metadados após a purga)
20. Ecrã de configuração: token, chat ID, formato da mensagem
21. Onboarding: permissões uma de cada vez ([permissoes.md](permissoes.md) §3)

## Fase 7 — Testes e robustez ⬜

22. Testes unitários do repositório, regras puras e worker
23. Toda a [matriz de testes manuais](desenvolvimento.md#testes-manuais) passa no dispositivo real
24. Alguns dias de uso real antes de confiar na app

## Fase 8 — Distribuição ⬜

25. Keystore de release e assinatura ([desenvolvimento.md](desenvolvimento.md#build-de-release-assinado))
26. Build de release e instalação por sideload

## Fora do âmbito

- Backend, múltiplos dispositivos, múltiplos canais ou roteamento condicional — ver "Caminho de evolução" em [arquitectura.md](arquitectura.md) §5
- Play Store
- `READ_SMS` e recuperação a partir da caixa de entrada
