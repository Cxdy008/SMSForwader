## Description

<!-- Clearly describe what this PR does and why. Link the related issue: Closes #... -->

## Type of Change

- [ ] Bug fix
- [ ] New feature
- [ ] Documentation
- [ ] Refactor / chore
- [ ] Other

## Screenshots (if UI)

<!-- Add before/after screenshots or screen recordings for UI changes. -->

## How to Test

<!-- Steps for a reviewer to verify the change locally (e.g. `adb emu sms send <number> <text>` on an emulator). -->

1.
2.
3.

## Checklist

- [ ] My branch is up to date with `main` and has a focused, single concern
- [ ] `./gradlew test lint assembleDebug` passes locally
- [ ] No hard-coded user-facing strings — all strings are in `res/values/strings.xml`
- [ ] No bot token, chat ID or other credential is committed (tokens are stored only via encrypted storage)
- [ ] I have not introduced a backend or intermediate server (see `docs/arquitectura.md`, "Caminho de evolução")
- [ ] If this changes data flow, the forwarding channel, or the retry/queue layer, `docs/arquitectura.md` is updated in this same PR
- [ ] The safety contract in `docs/pipeline.md` §2 still holds (no SMS content or token in logs, nothing marked sent before a 200 OK)
- [ ] Phase status in `docs/roadmap.md` is updated if this PR completes a step
- [ ] I have updated other documentation if needed (`README.md`, `CLAUDE.md`)
- [ ] I have linked the related issue

## Related Issue

Closes #
