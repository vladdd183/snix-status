# snix build oci: стабильность и поломки

Вердикт: в прод не годен. Это не отдельная команда `snix build oci`, а Linux-only реализация `BuildService` с адресом `oci:<каталог бандлов>`. Каркас в монорепо есть: OCI runtime spec, bundle, вызов `runc`, FUSE-входы, забор выходов в castore. Надёжной формы нет. Дефолт и у демона, и у eval — `dummy:`, то есть сборка по умолчанию ничего не исполняет. Открытые issue закрывают fixed-output с сетью, checkPhase вроде coreutils и чтение файлов с правами `0600`. Preflight нет. Живой прогон `runc` в этом срезе не запускался: ниже только код, документация и issues на `https://git.snix.dev/snix/snix`.

Этот файл не заменяет `docs/snix-status.md` и не переписывает матрицу «что в Snix работает».

## Срез

Монорепо склонирован с `https://git.snix.dev/snix/snix`. HEAD этого среза: `ed7d6eb0fa7d240cc7a6a4bae25e0ce4f6297b62` (2026-10-06, `feat(castore): add GRPCBlobStoreWrapper`).

Пин предыдущего отчёта `docs/snix-status.md`: `f803bae05c19e59863f752bba4055d2c97d6ad4a` (2026-09-26, `refactor(build): use streaming builds everywhere`). Он предок HEAD. Между ними 66 коммитов. В diff этого диапазона по `snix/build` и `snix/cli/build` нет правок `oci/spec.rs`, `oci/bundle.rs`, `oci/subuid.rs`, `oci/mod.rs`. Поведенческий diff OCI-сервиса — переименование `BlobService` → `BlobEngine` в `snix/build/src/buildservice/oci.rs` (коммиты `972088e` и `f312106`, оба 2026-10-06). Семантика песочницы с пина отчёта не сдвинулась.

Правило пустого прода то же, что в `docs/snix-status.md`: `web/content/about.md` говорит, что API не стабильны и drop-in замены Nix нет.

## Что это за «команда»

Строки `snix build oci` в дереве нет.

- Обёртка `snix` ищет бинарник `snix-{подкоманда}`: `snix/cli/base/src/lib.rs`, `find_command`. `snix build` — это исполняемый файл `snix-build`, не глагол «собрать».
- У `snix-build` одна подкоманда, `daemon`. Адрес сервиса — флаг `--build-service-addr`, по умолчанию `dummy:` (`snix/cli/build/src/main.rs`).
- Eval ходит в тот же `from_addr` и тоже по умолчанию `dummy:` (`snix/cli/eval/src/args.rs`, `snix/cli/eval/src/lib.rs`). Комментарий CLI: интерфейс не стабилен, ни одно хранилище не пишет в локальный `/nix/store`.
- Схема `oci` принимается только на Linux и только как путь без authority: `oci:/var/tmp/bundles` да, `oci://...` нет (`snix/build/src/buildservice/from_addr.rs`). Рядом такие же схемы `bwrap:` и `grpc+*`. На не-Linux веток `oci` и `bwrap` в `match` нет.
- Документ компонента — четыре предложения: из `BuildRequest` делается OCI runtime spec, входы монтируются через snix-castore, затем вызывается `runc` (`web/content/docs/components/build/oci.md`, lastmod в шапке 2023-09-07). Гайда «как собрать деривацию через OCI» нет. `web/content/docs/guides/building.md` — это сборка самого Snix, не `snix build`.

Практический вызов, если бинарник вообще собран с `SNIX_BUILD_SANDBOX_SHELL`:

```
snix-build daemon --build-service-addr oci:/path/to/bundle-root
```

Либо тот же адрес в `--build-service-addr` у eval. Это не включено по умолчанию.

## Что путь делает

`OCIBuildService::do_build` (`snix/build/src/buildservice/oci.rs`):

1. Каталог бандла — `bundle_root/<uuid>`.
2. `make_spec(request, true, SANDBOX_SHELL)`. Второй аргумент зашит как `true`: rootless всегда. В `from_addr.rs` рядом TODO: «make sandbox shell and rootless_uid_gid».
3. `SANDBOX_SHELL` — это `env!("SNIX_BUILD_SANDBOX_SHELL")`, значение на этапе компиляции, не флаг запуска.
4. `make_bundle` пишет `config.json`, каталоги `inputs`, `root`, `scratch`.
5. Входы — FUSE: `FuseDaemon` на `bundle/inputs`, 4 потока, `list_root: true`.
6. `runc run --bundle <path> <name>`, stdout/stderr в пайпы, stdin закрыт, `kill_on_drop`.
7. Дальше общий `run_build_streaming`: ненулевой код — `BuildFailure::NonzeroExitCode`, затем ingestion выходов в castore с reference scan.

Одновременно не больше двух сборок. Константа `MAX_CONCURRENT_BUILDS = 2` с комментарием, что иначе кончаются файловые дескрипторы, и TODO «make configurable».

Спека (`snix/build/src/oci/spec.rs`):

- Сеть по умолчанию в своём network namespace. Если в ограничениях есть `NetworkAccess`, namespace сети не создаётся (сеть хоста) и в контейнер rbind-ятся `/etc/resolv.conf`, `/etc/services`, `/etc/hosts` только на чтение.
- `ProvideBinSh` монтирует зашитый `SANDBOX_SHELL` на `/bin/sh`.
- Процесс: uid 1000, gid 100, `no_new_privileges`, rlimit nofile 1024/1024, hostname `localhost`, cwd от `working_dir`.
- User namespace и маппинг id берутся из `/etc/subuid` и `/etc/subgid` эффективной учётки. Нужна хотя бы одна подчинённая пара. Иначе `SubordinateError::MissingEntry` ещё до `runc`.
- В rootless-наборе capabilities шире, чем в корневом (chown, dac_override, mknod, setuid и другие). OCI всегда вызывает rootless-ветку.
- Cgroup namespace выключен комментарием: на «сломанном» cgroup хоста он ломает старт.

Выходы ищутся только внутри scratch-каталогов бандла (`OciOutputs::find_path`). Чужой путь — `None`, дальше `BuildFailure::MissingOutput`.

Glue, который превращает деривацию Nix в `BuildRequest` (`snix/build-glue/src/builder/mod.rs`), всегда ставит `ProvideBinSh`, scratch `build` и `nix/store`, рабочий каталог `build`. Fixed-output добавляет `NetworkAccess`. Это общий путь и для OCI, и для bwrap: различие только в том, какой адрес передали в `from_addr`.

## Что уже держится

Это каркас, не зелёный прогон nixpkgs.

- Разбор URI `oci:` покрыт тестом `from_addr`: пустой путь отвергается, путь без authority принимается, authority отвергается. Тесты под `#[cfg(target_os = "linux")]`.
- Разбор первой строки `/etc/subuid` покрыт юнит-тестами на временном файле (`oci/subuid.rs`). Сам `for_effective_user()` против живых `/etc/subuid` тестом не закреплён.
- Имена scratch и поиск выхода внутри них покрыты тестами `oci/mod.rs`.
- Потоковый протокол `DoBuild` → `stream BuildUpdate` есть в `snix/build/protos/rpc_build.proto`. Комментарий в proto: `BuildRequest` ещё «some sort of spec», снаружи его планируют обернуть.
- Документ компонента подтверждает ту же тройку: spec, castore mount, `runc`.

Чего в крейте `snix-build` нет: теста, который поднимает `runc` и доводит `BuildRequest` до ingestion. Поиск `#[test]` / `#[tokio::test]` в `snix/build` даёт разбор URI, subuid, scratch-пути, bwrap-хелперы и стрим процесса. Интеграционного OCI-прогона нет.

## Классы поломок

### Хост не подготовлен, и это не проверяется на старте

Issue 131 открыт с 2025-05-05, обновлён 2025-05-05: «OCI builder: add preflight checks». Автор перечисляет допущения, которые сегодня молча становятся ошибкой уже в сборке: suid `fusermount`, `allow_other`, `runc` в `PATH`, user namespace и `/etc/subgid`. В `OCIBuildService::new` этих проверок нет: конструктор только кладёт путь и семафор. `runc` ищется в момент `Command::new("runc")`. Subuid читается внутри `make_spec`, то есть на каждый билд.

Типичные отказы этого класса: нет бинарника `runc`, нет строки в `/etc/subuid`/`/etc/subgid`, FUSE не стартовал (`failed to start fuse daemon`).

### Fixed-output и сеть

Glue для деривации с fixed outputs добавляет `NetworkAccess`. Спека тогда не изолирует сеть и монтирует три файла из `/etc`.

Issue 155 открыт с 2025-07-07: «Fixed-output derivations fail to build in oci builder». Лог `runc` из issue: `error mounting "/etc/resolv.conf" ... flags: 0x1021: operation not permitted`. В тексте issue опции указаны как `bind,ro`. В текущем `spec.rs` те же три пути монтируются типом `none` и опциями `rbind,ro`. Файл с пина `f803bae` не менялся, issue не закрыт, повторного лога в этом срезе нет. Считать починкой смену `bind` → `rbind` нельзя: доказательства успешного FOD нет. Класс остаётся открытым.

### Нет `/etc/passwd`, checkPhase врёт

`bundle.rs` создаёт `root/etc` и оставляет TODO: «populate /etc/{group,passwd}. It's a mess?»

Issue 110 открыт, обновлён 2025-10-03: «[OCI] coreutils checkPhase tests fail in snix build sandbox». В теле прямой лог:

```
whoami: cannot find name for user ID 1000: No such file or directory
FAIL: whoami
```

Там же падают `tests/rm/cycle`, `tests/chmod/no-x`, `tests/rm/isatty`, `tests/rm/empty-inacc`. Это не разовый флак: ожидаемый stderr coreutils не совпадает с тем, что даёт эта песочница. Пользователь 1000 зашит в `configure_process`, имени ему никто не выдаёт.

### Structured attrs и какой `/bin/sh` зашили

`build-glue` кладёт `__structuredAttrs` в `/build/.attrs.sh` и `/build/.attrs.json`, срезая ведущий слэш (`structured_attrs.rs`: пути константы `/build/.attrs.sh` и `/build/.attrs.json`, в map попадает `[1..]`). Комментарий в том же файле: structured attrs работают только с настоящим bash, потому что `NIX_ATTRS_SH_FILE` использует `declare`, а в busybox sandbox shell его нет.

Какой бинарник окажется на `/bin/sh`, решает не запрос, а компиляция `snix-build`:

- crane-обвязка `snix/default.nix` ставит `SNIX_BUILD_SANDBOX_SHELL = "/homeless-shelter"`.
- crate2nix override `snix/utils.nix` на Linux ставит `pkgs.pkgsStatic.busybox + "/bin/sh"`.
- devshell `snix/shell.nix` экспортирует `busybox-sandbox-shell` как `/bin/busybox`.

Три разных значения на один и тот же `env!`. Бинарник, собранный crane-путём, примонтирует в контейнер `/homeless-shelter`. Это не флейк рантайма, это развилка сборки самого Snix.

### Additional files только если scratch называется `build`

`bundle.rs`, `TODO(#152)`: дополнительные файлы пишутся лишь когда scratch-путь равен `build`, и каждый путь обязан начинаться с компоненты `build`. Иначе ошибка «Additional files must start with build/». Issue 152 закрыт 2026-04-26 («Improve scratch path and additional files handling in OCI builds»), но комментарий и ветка `if p == Path::new("build")` на HEAD живы. Для текущего glue это совпадает с тем, что glue всегда передаёт scratch `build`. Для любого другого `BuildRequest` — нет. Issue 205 (закрыт 2026-03-25) фиксировал ту же путаницу ведущего слэша; glue теперь слайс `[1..]`, OCI по-прежнему требует префикс `build/` без абсолютного корня.

### Ingestion не читает файлы песочницы

Issue 156 открыт с 2025-07-07: «Builder ingestion can fail if the build produces files with restricted permissions». Пример — `python312Packages.flit-core.dist`, файл `0600` владельца `100000`. После `runc` ingestion идёт от пользователя сервиса, не от uid 1000 внутри userns (`streaming.rs` → `ingest_path`). Если выход не читается хостом, сборка уже с кодом 0 всё равно падает на ingestion. Issue сформулирован про builder вообще; OCI попадает в него тем же `run_build_streaming`, что и bwrap.

### bwrap — не запасной OCI

Рядом есть `bwrap:`. Issue 211 (открыт 2026-04-26) описывает лишнюю группу 65534 у непривилегированного bubblewrap и поломку `chgrp.test` rsync. Это другой билдер. Путать с логом OCI нельзя: OCI мапит uid через subuid, bwrap — через флаги `bwrap`.

### nix-daemon этот путь не вызывает

`snix/nix-daemon/src/lib.rs`: `build_paths` возвращает `Ok(())` и аргументы не смотрит. `build_derivation` и `build_paths_with_results` возвращают «Operation … is not implemented». Демон Nix-протокола до `OCIBuildService` не доходит. Срыв сборки, в котором Nix-демон станции отвалился по таймауту substituter, к этому коду не относится и в вердикт OCI не входит.

## Воспроизводимость

Удачные выходы ingestion пишет в castore по содержимому (`ingest_path` + reference scanner). Это не делает саму песочницу воспроизводимой.

Мешают, и это видно из кода, а не из догадки:

- `/bin/sh` зависит от того, каким Nix-входом собрали `snix-build` (три значения выше).
- FOD тащит внутрь хостовые `resolv.conf` / `hosts` / `services`. Даже если issue 155 закроют, эти файлы — снимок хоста.
- `/etc/passwd` пуст, uid 1000 безымянный. То, что в nixpkgs завязано на имя пользователя, расходится с Nix.
- rlimit nofile зашит в 1024. Пакеты, которые ждут больший предел, ведут себя иначе, чем в песочнице Nix.
- Cgroup namespace выключен, учёта ресурсов нет.
- `terminal: true` в process spec.
- Комментарий `scratch_name` обещает lower-hex blake3, код делает BASE32 (`oci/mod.rs`). На байты выхода это не влияет, на чтение кода — ловушка.

Отдельного теста «два прогона одной деривации дали один и тот же castore node» для OCI нет.

## Флейки

Зафиксированного плавающего OCI-теста в дереве нет: интеграционного теста тоже нет, поэтому «зелёный/красный CI OCI» как ряд не существует.

То, что снаружи выглядит как флак, в коде является развилкой хоста:

- семафор на 2 из-за нехватки fd — под нагрузкой сборка ждёт, а не ретраится;
- FUSE и `runc` падают по-разному на машине без `allow_other`, без subuid или без `runc` в `PATH` (issue 131);
- issue 110 — устойчивое расхождение checkPhase, не intermittent.

Issue про CI Snix (198, 200: binary cache и git-кэш билдеров) к стабильности OCI-рантайма не привязаны. В них нет лога `runc`.

## Открытые issue, которые относятся к OCI

- 131 — preflight (`fusermount`, `allow_other`, `runc`, subgid). Открыт.
- 155 — FOD, `runc` не монтирует `/etc/resolv.conf`. Открыт. Код с тех пор сменил опции на `rbind`, повторного успешного лога нет.
- 110 — coreutils checkPhase, в том числе `whoami` для uid 1000. Открыт. Совпадает с TODO про `/etc/passwd` в `bundle.rs`.
- 156 — ingestion файлов `0600` владельца песочницы. Открыт. Общий streaming-путь.
- 157 — «evaluate switching from runc to youki». Открыт с 2025-07-07. В `spawn_bundle` по-прежнему `Command::new("runc")`. Замены нет.
- 211 — группа 65534 у bwrap. Открыт. Не OCI, рядом, чтобы не смешать.

Закрытые, которые код ещё помнит: 152 (scratch/additional files) закрыт, `TODO(#152)` и ветка `p == "build"` остаются. 86 (subuid) и 87 (additional files не монтировались) закрыты; subuid в спеке есть, монтирование additional files остаётся узким случаем выше. 205 закрыт на стороне ведущего слэша в glue.

## Старые срезы

В каталоге `meta-intent/corpus/sb/work/tech/nix-ipfs` (файлы, на которые указывает карточка: `research/a10-tvix-vs-snix-fork.md`, `research/a8-tvix-snix-ipfs-substituter.md`, `synthesis/S3-replit-failures-snix-fork-ipfs-swh.md`) строк `build oci` и `stage B4` нет. Стадию B4 как факт этот отчёт не использует. Расхождения августа 2026 по всему Snix уже разобраны в `docs/snix-status.md`; сюда они не копируются.

Что из тех срезов прямо касается этого пути:

- Утверждение a10, что drop-in Nix нет, с OCI не расходится: `about.md` говорит то же, nix-daemon сборку не реализует, дефолт билдера — `dummy:`.
- Матрица `docs/snix-status.md` помечает весь `snix-build` как `эксперимент` и напоминает, что OCI/bwrap только под Linux. Этот срез уже: внутри эксперимента OCI — каркас с открытыми классами поломок, не «эксперимент, которым уже можно собирать nixpkgs».

## Насколько далеко до надёжной формы

Близко только описание желаемого цикла: request → spec → bundle → runc → ingestion. Далеко всё, без чего этот цикл нельзя поставить на чужую машину и на nixpkgs:

- адрес не дефолт, подкоманды `oci` нет, гайд не обновлялся с 2023-09-07;
- нет preflight (issue 131) и нет интеграционного теста с `runc`;
- FOD с сетью не подтверждён после issue 155;
- checkPhase, которым nixpkgs проверяет coreutils, уже красный (issue 110);
- `/bin/sh` зашит при сборке Snix и для structured attrs часто недостаточен;
- ingestion ломается на типичных правах wheel (issue 156);
- лимит две сборки и выключенный cgroup — ручные потолки, не политика;
- youki так и остаётся идеей (issue 157), зависимость от `runc` в коде.

Пока эти пункты открыты, класс такой: эксперимент для чтения и точечных проб на подготовленном Linux с subuid, FUSE и `runc`. Не прод и не замена песочницы Nix.
