# Воспроизводимая dev-среда: Homebrew, Nix, Docker, Railway и Terraform

Воспроизводимое окружение собирается слоями, но слои не надо включать все подряд: **фиксируй каждый уровень, который реально участвует в проекте, и не добавляй следующий ради красоты**. Для обычного coding-проекта на macOS здоровый дефолт — Homebrew для пользовательской машины, Nix flakes для системного/dev-слоя проекта, lock-файлы языка для зависимостей приложения, Docker только для сервисов или runtime-изоляции, VM только для другой ОС/kernel или недоверенного кода. Railway/Render/Fly/Heroku-подобные PaaS — не слой «между Docker и Terraform», а managed shortcut: они забирают себе часть операционного риска, который в self-managed варианте остаётся у тебя. Terraform стоит отдельно: он не про пакеты и не про runtime, а про внешнюю инфраструктуру.

Эта заметка выросла из сессии [Nix vs Homebrew](https://chatgpt.com/share/6a3aa1b0-b168-83ed-8f26-258f7fb8a342). Смежный материал: [learning-web-development.md](learning-web-development.md) для Node/Vite/npm-слоя и [tool-packaging.md](tool-packaging.md) для выбора упаковки agent tools.

## 1. Карта слоёв

Главный вопрос не «какой инструмент моднее», а **что именно ты фиксируешь или изолируешь**.

```txt
Homebrew
  что нужно мне как пользователю Mac:
  GUI apps, casks, одноразовые CLI, macOS conveniences

nix-darwin
  как декларативно настроить сам Mac:
  системные packages, shell, launchd, defaults, user environment

Nix flake
  что нужно конкретному проекту на уровне dev/system:
  python, node, uv, pnpm, ffmpeg, openssl, terraform, jq

Language lockfiles
  что импортирует код приложения:
  uv.lock, pnpm-lock.yaml, package-lock.json, Cargo.lock, go.sum

Docker / Compose
  где и как запустить процесс или сервис:
  backend container, postgres, redis, worker, production-like runtime

Managed PaaS: Railway / Render / Fly / Heroku-like
  кто владеет build/deploy/runtime ops:
  service, env vars, domains, logs, metrics, restart policy, databases

VM
  какая машина и kernel:
  другая ОС, жёсткая граница ресурсов, недоверенный код

Terraform
  какие облачные ресурсы должны существовать:
  VPS, firewall, network, DNS, bucket, load balancer, IAM
```

Правило выбора:

```txt
Внешний бинарник / системная библиотека / CLI вокруг проекта → Nix
Библиотека, которую импортирует код → lock-файл языка
Сервис или runtime-граница → Docker Compose
Не хочешь владеть deploy/runtime ops → Railway/Render/Fly-like PaaS
Другая ОС/kernel или недоверенный код → VM
Облако / Hetzner / DNS / storage / networks → Terraform
Mac GUI / casks / пользовательские удобства → Homebrew
```

## 2. Homebrew и Nix на macOS: не надо начинать с войны

**На macOS Nix чаще надо добавлять к Homebrew, а не срочно заменять Homebrew.** Homebrew остаётся удобным слоем для машины: GUI/casks, macOS-native apps, быстрые одноразовые установки. Его собственная документация описывает `brew` как пакетный менеджер для UNIX-инструментов, которых нет в macOS, и как способ ставить pre-compiled casks; formula — это package definition, cask — package definition для подписанных upstream binary builds ([Homebrew manpage](https://docs.brew.sh/Manpage)).

Nix лучше ложится на проектный слой: «в этом репозитории должны быть доступны именно такие версии `python`, `node`, `uv`, `pnpm`, `ffmpeg`, `terraform`». Flakes — это unit для packaging Nix code reproducibly; `flake.nix` объявляет dependencies (`inputs`) и values, которые flake отдаёт наружу (`outputs`) ([Nix flake manual](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-flake)).

Практичная схема:

```txt
Homebrew:
  Docker Desktop, browsers, Raycast, Figma, Slack, TablePlus,
  одноразовые пользовательские CLI

Nix flakes:
  dev shells внутри проектов

nix-darwin:
  постепенно, когда хочется декларативно описать сам Mac
```

`nix-darwin` — это не «flake для проекта», а слой управления macOS через Nix. Проект описывает себя как Nix modules for Darwin и попытку принести декларативный system approach на macOS ([nix-darwin README](https://github.com/nix-darwin/nix-darwin)). Его стоит добавлять после того, как Nix уже доказал пользу на 1-2 проектах.

## 3. Почему одна утилита может стоять дважды

**Homebrew и Nix живут в разных мирах, поэтому одна и та же утилита может существовать в двух установках.** Например:

```txt
/opt/homebrew/bin/rg
  ripgrep из Homebrew

/nix/store/...-ripgrep/bin/rg
  ripgrep из Nix
```

Это нормальная цена параллельной схемы. Важно не перепутать: Nix обычно не кладёт пакет «в проект». Он кладёт immutable output в общий `/nix/store`, а dev shell проекта добавляет нужные store paths в `PATH` и другие переменные окружения.

Между Nix-проектами дублирование не обязательно. Если два проекта ссылаются на один и тот же `nixpkgs` commit и получают один и тот же derivation, store path будет переиспользован. Если `flake.lock` разный, options разные или версия пакета другая, появятся разные store paths.

Модель:

```txt
macOS user environment
├── Homebrew: /opt/homebrew/bin/node
└── Nix store: /nix/store/...-nodejs/bin/node

project-a/flake.nix → подмешивает node из Nix в dev shell
project-b/flake.nix → может подмешать тот же store path или другой
```

## 4. `inputs`, `outputs` и `flake.lock`

**`flake.nix` — это контракт: что flake потребляет и что отдаёт наружу.** `inputs` — зависимости flake-уровня, `outputs` — Nix-значения, которые flake предоставляет. В мануале Nix `inputs` описаны как attrset зависимостей flake, а `outputs` — как функция, которая получает outputs входных flakes и возвращает values этого flake ([Nix flake format](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-flake)).

Минимальный пример:

```nix
{
  description = "Project dev environment";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

  outputs = { self, nixpkgs }:
    let
      system = "aarch64-darwin";
      pkgs = import nixpkgs { inherit system; };
    in {
      devShells.${system}.default = pkgs.mkShell {
        packages = with pkgs; [
          python312
          uv
          nodejs_22
          pnpm
          ffmpeg
          jq
        ];
      };
    };
}
```

Здесь:

```txt
input:
  nixpkgs = источник Nix-кода/пакетов

output:
  devShells.aarch64-darwin.default = окружение, в которое входит nix develop
```

`input` — не только «каталог пакетов». Это любой внешний flake / репозиторий / источник Nix-кода: `nixpkgs`, `nix-darwin`, `home-manager`, `flake-utils`, приватный company flake. Просто чаще всего первым input становится `nixpkgs`.

`flake.lock` — близкий аналог `Podfile.lock`, `package-lock.json` или `uv.lock`, но на Nix-уровне:

```txt
flake.nix
  хочу nixpkgs из nixos-unstable

flake.lock
  конкретный commit nixpkgs = ...
```

Команда `nix flake lock` создаёт недостающие entries в `flake.lock` и держит lock для каждого input, указанного в `flake.nix`; для обновления существующих entries используется `nix flake update` ([nix flake lock](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-flake-lock)).

Важная граница: `flake.lock` фиксирует Nix inputs, а не обязательно каждую Python/Node-библиотеку приложения.

## 5. Dev/system layer против application dependencies

**Nix хорошо фиксирует dev/system layer; language package managers хорошо фиксируют application dependencies.** Это два вложенных уровня, и путаница между ними создаёт хаос.

Dev/system layer — внешние инструменты, рантаймы, CLI и системные библиотеки:

```txt
python
node
uv
pnpm
terraform
postgresql client
redis
sqlite
openssl
pkg-config
ffmpeg
imagemagick
ripgrep
jq
git
gh
```

Ты обычно не импортируешь их как библиотеку приложения. Они нужны, чтобы проект собирался, запускался или разрабатывался.

Application dependencies — то, что код импортирует или что тесно живёт внутри экосистемы проекта:

```txt
Python: fastapi, pydantic, sqlalchemy, openai, pytest
Node: react, vite, express, zod, typescript, eslint
Rust: serde, tokio, axum
Go: modules из go.mod
```

Их обычно фиксируют lock-файлы языка:

```txt
Python → uv.lock / poetry.lock / requirements lock
Node   → package-lock.json / pnpm-lock.yaml / yarn.lock / bun.lock
Rust   → Cargo.lock
Go     → go.sum
Ruby   → Gemfile.lock
```

Документация uv формулирует это ровно на своём уровне: locking resolves project dependencies into a lockfile, syncing installs packages from the lockfile into the project environment; `uv run` автоматически lock/sync'ает проект перед запуском команды ([uv locking and syncing](https://docs.astral.sh/uv/concepts/projects/sync/)).

## 6. Куда класть спорные инструменты

**Иногда один и тот же инструмент можно положить и в Nix, и в lock-файл языка; выбирай по тому, к чему он ближе.**

Правило:

```txt
Пакет импортируется кодом приложения → language lockfile
Generic внешний binary/CLI → Nix
CLI тесно связан с framework plugins/config/scripts → language lockfile
```

Примеры Node:

| Штука | Где чаще фиксировать |
|---|---|
| `react`, `express`, `zod` | `package.json` + lock |
| `typescript` | чаще `package.json`, потому что IDE/scripts |
| `eslint` | чаще `package.json`, из-за plugins/config |
| `prettier` | можно оба; чаще `package.json`, если часть project scripts |
| `node` | Nix |
| `pnpm` | Nix или Corepack; для строгого shell — Nix |
| `ffmpeg`, `jq`, `terraform` | Nix |

Примеры Python:

```txt
uv.lock:
  fastapi, sqlalchemy, pytest, ruff, mypy, alembic

flake.nix:
  python312, uv, postgresql, libpq, openssl, ffmpeg, pkg-config
```

`ruff`, `black`, `pytest`, `mypy` — спорная зона: это Python-пакеты и dev tools одновременно. Если они должны видеть те же зависимости, что проект, запускаться из `pyproject.toml` и работать в CI как часть Python-проекта, держи их в `uv.lock`. Если инструмент внешний и не зависит от Python project environment, его можно держать в Nix.

Плохой симптом — одна версия `ruff` в `.venv`, другая в `PATH` из Nix, третья в CI. Это не воспроизводимость, а лотерея.

## 7. Как входить в окружение проекта

**`nix develop` включает Nix-часть окружения; language lockfiles синхронизируются отдельно.** Команда `nix develop` запускает interactive build environment; если output явно не указан, она пробует `devShells.<system>.default`, затем `packages.<system>.default` ([nix develop](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-develop)).

Проект со смешанным стеком может выглядеть так:

```txt
my-project/
├── flake.nix
├── flake.lock
├── .envrc
├── pyproject.toml
├── uv.lock
├── package.json
├── pnpm-lock.yaml
└── docker-compose.yml
```

Первичный setup:

```bash
nix develop
uv sync
pnpm install
docker compose up -d
```

`nix develop` не обязан запускать `uv sync` и `pnpm install`. Обычно sync/install делают при первом входе, после изменения lock-файлов, после pull из git или в CI. В `shellHook` лучше выводить подсказку, а не автоматически менять `.venv` и `node_modules` на каждый вход.

Удобный слой поверх — `direnv`: это shell extension, который загружает и выгружает переменные окружения в зависимости от текущей директории ([direnv](https://direnv.net/)). В `.envrc` можно написать:

```bash
use flake
```

`direnv` stdlib описывает `use flake` как загрузку build environment derivation similar to `nix develop`, по умолчанию из `flake.nix` текущей папки ([direnv stdlib](https://direnv.net/man/direnv-stdlib.1.html)).

Командный UX после первичного `direnv allow`:

```bash
cd my-project   # Nix dev shell включился автоматически
uv sync         # когда надо
pnpm install    # когда надо
just dev
```

## 8. Docker: контейнер, не VM, и не просто песочница

**Docker отвечает за runtime boundary: где и как запустить процесс в изолированной среде.** Это не то же самое, что Nix, но они пересекаются в задаче «получить управляемое окружение».

Docker docs проводят границу так: VM — это целая ОС со своим kernel, drivers, programs and applications; container — isolated process with all files it needs to run, и несколько containers share the same kernel ([Docker: containers vs VMs](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)).

На Linux контейнеры используют Linux kernel хоста. На macOS Docker Desktop использует виртуализационный слой; FAQ Docker Desktop for Mac описывает HyperKit как hypervisor поверх macOS Hypervisor.framework и хранение Linux containers/images в отдельном disk image file, что отличается от Docker on Linux ([Docker Desktop for Mac FAQ](https://docs.docker.com/desktop/troubleshoot-and-support/faqs/macfaqs/)). Поэтому точная формула такая:

```txt
Docker container ≠ VM

На macOS:
  Docker Desktop поднимает Linux VM/virtualized environment,
  а уже внутри него запускаются Linux containers.
```

Docker — не только «песочница». Он ещё:

```txt
упаковывает runtime filesystem
запускает процесс одинаково на разных машинах
даёт service composition через Compose
приближает локальный запуск к production
помогает delivery/deploy
```

Но Docker не автоматически даёт строгую воспроизводимость. Docker tag mutable: publisher can update a tag to point to a new image; `FROM alpine:3.21` сегодня и через три месяца может резолвиться в разные patch versions. Для строгой фиксации Docker рекомендует pin to digest ([Docker build best practices](https://docs.docker.com/build/building/best-practices/)); digest — immutable SHA-256 identifier exact image content ([Docker image digests](https://docs.docker.com/dhi/core-concepts/digests/)).

Слабая фиксация:

```Dockerfile
FROM node:22
RUN apt-get update && apt-get install -y python3 ffmpeg openssl
```

Более строгая фиксация:

```Dockerfile
FROM node:22.11.0-bookworm@sha256:...
RUN apt-get update && apt-get install -y \
    python3=3.11.2-1+b1 \
    ffmpeg=7:5.1.6-0+deb12u1 \
    openssl=3.0.16-1~deb12u1
```

`apt-get install pkg=version` — официальный синтаксис apt-get; Debian manpage прямо говорит, что specific version selects by following package name with equals and version ([apt-get manpage](https://manpages.debian.org/bookworm/apt/apt-get.8.en.html)). Но версии должны оставаться доступными в репозитории; для долгой воспроизводимости нужны snapshots или другой механизм pinning.

## 9. Nix и Docker вместе

**Nix отвечает за “из чего состоит окружение”, Docker — за “где исполняется процесс”.** Поэтому они могут использоваться отдельно или вместе.

```txt
Только Nix:
  локальная разработка в обычной папке macOS

Только Docker:
  контейнер, где зависимости ставятся через apt/apk/pip/npm/uv

Docker + Nix:
  контейнер/runtime boundary,
  а пакеты внутри собираются или подтягиваются через Nix
```

Nix официально можно запускать внутри Docker: мануал Nix показывает `docker run -ti docker.io/nixos/nix`, а также указывает, что официальный Docker image Nix создан через `pkgs.dockerTools.buildLayeredImage` ([Using Nix within Docker](https://nix.dev/manual/nix/latest/installation/installing-docker)).

Обратная связка тоже возможна: в Nix dev shell кладёшь Docker CLI / Compose, а сами сервисы запускаешь через Compose:

```txt
flake.nix:
  python312, uv, nodejs_22, pnpm, ffmpeg, docker, docker-compose, just

docker-compose.yml:
  postgres, redis, maybe backend runtime
```

Рабочая команда:

```bash
nix develop
uv sync
pnpm install
docker compose up -d
just dev
```

## 10. Не “все уровни по дефолту”, а минимальный достаточный набор

**Воспроизводимость — не лестница, где всегда нужен VM + Docker + Nix + language locks.** Нужен минимальный набор, закрывающий реальные риски проекта.

```txt
Простой Python CLI:
  uv.lock
  возможно .python-version / managed Python через uv
  Nix не обязателен

Python CLI, который надо повторить через месяц на другой машине:
  flake.nix + flake.lock
  uv.lock

Backend + frontend + Postgres + Redis:
  flake.nix + flake.lock
  uv.lock
  pnpm-lock.yaml
  docker-compose.yml

Недоверенный AI-generated проект:
  VM
  Docker/Compose
  Nix
  language lockfiles
```

Для простых Python-утилит `uv` может быть достаточно. uv создаёт project environment в `.venv`, `uv run` запускает команду в этом окружении, `uv sync` явно создаёт/синхронизирует его ([uv project structure](https://docs.astral.sh/uv/concepts/projects/layout/)). uv умеет находить system Python и устанавливать managed Python; конкретную версию можно запросить через `--python`, и uv скачает её при необходимости ([uv Python versions](https://docs.astral.sh/uv/concepts/python-versions/)).

Nix становится оправдан, когда надо фиксировать не только Python-пакеты, но и системный слой вокруг них:

```txt
конкретный Python binary
openssl/sqlite/zlib/libffi/pkg-config
ffmpeg/imagemagick/tesseract
Postgres/libpq
Node frontend рядом
Terraform/kubectl/other CLI
одинаковый dev shell на нескольких машинах
```

## 11. Railway и PaaS: кто владеет риском

**Railway — это managed PaaS shortcut, а не «ещё один слой между Docker и Terraform».** Он заменяет для пользователя операционную связку `VPS + container runtime + deploy pipeline + env/logs/domains/db wiring`; под капотом Railway всё равно опирается на те же низкоуровневые идеи — containers, images, orchestration, networking, storage, provisioning. Разница не в магии, а в ответе на вопрос: **кто владеет риском?**

Railway service в официальных docs определён как deployment target; под капотом services — containers, deployed from an image. Service может деплоиться из GitHub repo, local directory или Docker image; если в source repo найден Dockerfile, Railway автоматически использует его для build'а image ([Railway Services](https://docs.railway.com/services)). Если Dockerfile нет, Railway может собрать приложение через Railpack: он анализирует код и генерирует optimized container images ([Railway Builds](https://docs.railway.com/builds)).

Модель:

```txt
Self-managed путь:
  Nix/uv/pnpm → Dockerfile/image
  Terraform → VPS/network/firewall/db resources
  Docker Compose/systemd/Caddy/scripts → runtime/deploy/logs/restarts

Railway путь:
  Nix/uv/pnpm локально
  GitHub repo / local directory / Docker image → Railway service
  Railway → build, deploy, runtime, env vars, domains, logs, metrics, db service
```

Railway не заменяет Docker как концепцию: Railway service всё равно контейнерный runtime target. Но Railway может заменить тебе обязанность писать и обслуживать Dockerfile/Compose для простого приложения, потому что сам соберёт image из кода или примет готовый Docker image.

Railway не заменяет Terraform вообще: Terraform управляет произвольной инфраструктурой — Hetzner, AWS, Cloudflare, networks, buckets, IAM, DNS. Railway заменяет Terraform только в узком практическом смысле: «мне не надо самому создавать VPS/firewall/network/deploy pipeline для этого приложения».

### Почему “агент + CLI” не делает Railway бессмысленным

LLM снижает ценность простого интерфейса: агенту несложно написать `Dockerfile`, `docker-compose.yml`, Terraform, deploy scripts и команды для VPS. Но агент заменяет неудобный человеческий интерфейс, **а не платформу, state и operational responsibility**.

Если агент запускает Railway CLI, Docker CLI или Terraform CLI, CLI — лишь интерфейс. Вопрос в том, что остаётся source of truth:

```txt
Плохо:
  агент выполнил railway/docker/ssh команды один раз,
  state живёт только в платформе и истории чата

Лучше:
  Dockerfile
  docker-compose.yml
  terraform/*.tf
  railway.toml / railway.json, если выбран Railway
  .env.example
  README deploy/runbook
```

Railway поддерживает config-as-code: по умолчанию ищет `railway.toml` или `railway.json` рядом с кодом; build/deploy настройки из файла применяются к текущему deployment и override'ят dashboard values ([Railway Config as Code](https://docs.railway.com/config-as-code/reference)). Это не Terraform: файл не описывает всю инфраструктуру платформы, но убирает часть «магии UI/CLI» из deploy-конфигурации.

### Чек-лист ops не равен managed ops

Агент + чек-лист может закрыть setup простого VPS:

```txt
firewall rules
SSH hardening
Docker/Compose setup
systemd restart policies
log rotation
disk usage checks
backup script
restore command
TLS через Caddy
security updates
basic monitoring
```

Но checklist сам по себе не закрывает постоянное владение риском:

```txt
backup реально выполняется каждый день
ошибки backup видны
restore проверен
disk space не кончился
security updates не сломали runtime
alerts доходят
restart policy работает
секреты и доступы не протухли
после ручной правки нет state drift
```

Поэтому формула такая:

```txt
Agent + checklist
  снижает стоимость setup.

Agent + automation + monitoring + runbook
  может заменить часть managed ops.

Railway/PaaS
  покупает managed operational layer:
  меньше контроля и переносимости,
  меньше ежедневной ops-ответственности.
```

Выбор:

```txt
MVP / vibe-coded app / один web service + db:
  Railway нормально.

Простой проект, но нужен явный runtime:
  Railway + Dockerfile.

Долгоживущий проект, контроль цены, переносимость, infra-as-code:
  VPS + Docker/Compose + Terraform + agent-maintained runbook.

Деньги, клиентские данные, инциденты дороже времени:
  либо managed platform,
  либо собственный ops-layer как продукт, с мониторингом и restore drills.
```

## 12. Terraform: инфраструктура, не окружение приложения

**Terraform описывает внешнюю инфраструктуру как код; Nix может установить `terraform` binary, но Terraform-файлы управляют облачными ресурсами.** HashiCorp описывает Terraform как infrastructure as code tool для build, change, and version cloud and on-prem resources; он управляет compute, storage, networking, DNS entries and SaaS features через providers ([What is Terraform](https://developer.hashicorp.com/terraform/intro)).

Место Terraform в карте:

```txt
flake.nix
  в dev shell должен быть terraform

infra/main.tf
  в Hetzner/AWS/GCP/Cloudflare должны существовать такие ресурсы

.terraform.lock.hcl
  зафиксированы Terraform providers
```

Типовой workflow:

```bash
terraform init
terraform plan
terraform apply
```

`terraform plan` создаёт execution plan: читает текущее состояние, сравнивает его с configuration и предлагает actions, которые приведут remote objects к конфигурации; сам `plan` изменения не применяет ([terraform plan](https://developer.hashicorp.com/terraform/cli/commands/plan)). Apply применяет предложенные операции после approval как часть core workflow Write → Plan → Apply ([Terraform workflow](https://developer.hashicorp.com/terraform/intro)).

Аналогия:

```txt
Dockerfile
  описывает runtime/container image

flake.nix
  описывает локальный dev/system layer

Terraform .tf files
  описывают cloud/on-prem infrastructure
```

## 13. Terraform на Hetzner

**На Hetzner Terraform полезен, если ты используешь Hetzner Cloud resources, а не просто один раз вручную созданный VPS.** У Hetzner Cloud есть provider `hetznercloud/hcloud`; его docs говорят, что provider interacts with resources supported by Hetzner Cloud и требует API token, который можно передать через `token` или `HCLOUD_TOKEN` ([hcloud provider docs](https://raw.githubusercontent.com/hetznercloud/terraform-provider-hcloud/main/docs/index.md)).

На Hetzner через Terraform обычно имеет смысл описывать:

```txt
servers
SSH keys
firewalls
private networks
volumes
floating IPs
load balancers
```

Для одного VPS Terraform может быть лишним: console + ssh + docker compose часто достаточно. Он окупается, когда есть staging/prod, firewall rules, volumes, private network, несколько серверов, регулярное пересоздание окружений или желание не забывать, что было накликано руками.

Разделение обязанностей:

```txt
Terraform:
  создать VPS, firewall, SSH key, volume, network

cloud-init / Ansible / Nix / shell:
  bootstrap системы внутри сервера

Docker Compose:
  поднять app, db, redis, caddy/nginx
```

## 14. Самая короткая формула

```txt
Homebrew ставит удобства на Mac.
nix-darwin описывает Mac декларативно.
Nix flakes фиксируют dev/system окружение проекта.
uv/pnpm/npm/cargo/go фиксируют зависимости приложения.
Docker запускает процессы и сервисы в контейнерах.
Railway/Render/Fly-like PaaS забирает часть deploy/runtime ops.
VM изолирует целую машину/kernel.
Terraform создаёт и меняет внешнюю инфраструктуру.
```

Дефолт для большинства isolated coding проектов:

```txt
Nix flake + language lockfiles
```

Добавь Docker, когда появились сервисы или runtime boundary. Выбери Railway/PaaS, когда хочешь купить managed ops вместо владения VPS. Добавь VM, когда появился недоверенный код или нужен другой kernel. Добавь Terraform, когда инфраструктура стала частью проекта, а не разовой ручной настройкой.
