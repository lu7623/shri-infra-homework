Деплой тут http://51.250.36.82:3000/hw/store
Тг @lu_agulova

# Домашнее задание ШРИ: Инфраструктура

Вам уже знакомо это приложение, вы работали с ним во время выполнения [домашнего задания по автотестам](https://github.com/dima117/testing-homework).
Теперь вам предстоит добавить в него CI/CD.

## Задача

Вам необходимо форкнуть этот репозиторий и добавить:
- сборку Docker-образа, который содержит работающее приложение;
- запуск тестов на каждый PR;
- процессы для отведения релизной ветки и внесения фиксов;
- деплой приложения на сервер.

Также вам будет нужно настроить виртуальную машину в Яндекс Облаке и [Container Registry](https://yandex.cloud/ru/docs/container-registry/quickstart/#registry-create), для этого вам выдадут промокоды на 1500 рублей.
IP-адрес этой машинки укажите в README, чтобы проверяющий мог открыть приложение, которое будет на ней запущено.

Репозиторий, который вы форкнули, должен быть публичным и настроен таким образом, чтобы никто не могу пушить ветку `main`, но любой человек мог:
- запускать релизные флоу,
- создавать пулл-реквесты и вливать их, если все проверки прошли успешно.

В файле `./src/client/pages/About.tsx` замените `[Your Name]` на своё имя и поправьте тест, которы проверяет содержимое этой строчки.

## Сборка образа

Вам необходимо подготовить Dockerfile и добавить скрипты в `package.json` для работы с Docker-образами.

Скрипт:
- `build:docker` — собирает образ (с тегом `shir-infra`), который будет запускать приложение
- `start:docker` — запускает последний собранный образа с тегом `shri-infra` на 3000 порту

## Как настроить сервер в Яндекс Облаке

Создайте виртуальную машину (Compute Cloud), воспользовавшись инструкцией по ссылке:\
https://yandex.cloud/ru/docs/compute/quickstart/quick-create-linux#create-vm

Рекомендуемые параметры:
- Операционная система: Ubuntu 20.04
- Диск: HDD на 10 Гб
- Вычислительные ресурсы: 2vCPU, 2 Гб RAM
- Включена опция «Публичный IP-адрес»

В итоге должно получиться чуть меньше 2 200 рублей в месяц — на две недели выполнения и проверки задания должно хватить.

## Описание работы процессов CI/CD

1. Флоу с проверками, который запускается при создании / изменении PR:

    - запускает параллельно линтер через `npm run lint` и тесты через  `npm run test`

    - нужно настроить ограничение на мердж изменений, если проверки не прошли

2. Флоу создания релиза, который запускается вручную (`on: [workflow_dispatch]`):

    - запускает параллельно линтер и тесты

    - здесь и дальше версией релиза будет считаться номер запуска флоу `${{ github.run_number }}`

    - отводит от `main` релизуню ветку `releases/<версия_релиза>`

    - [собирает docker-образ](https://yandex.cloud/ru/docs/container-registry/operations/docker-image/docker-image-create) с двумя тегами тегами:
        - `cr.yandex/<идентификатор_реестра>/app:<версия_релиза>`
        - `cr.yandex/<идентификатор_реестра>/app:<версия_релиза>_latest`
    
    - загружает docker-образ в Container Registry (необходимо, чтобы реджистри отображались оба тега)

    - создаёт тег, с номером текущей версии, который указывает на последний коммит в главной ветке

    - создаёт Issue в GitHub, которое содержит всю важную информацию:
        - дату,
        - автора релиза (тот, кто запустил флоу),
        - номер версии (`${{ github.run_number }}`),
        - список коммитов от предыдущего релизного (или фиксрелизного) тега,
        - ссылку на docker-образом в Yandex Container Registry\
          `cr.yandex/<идентификатор_реестра>/app:<версия_релиза>`

    - обновляет файл `CHANGELOG.md` в корне проекта, дописывает сверху новую версию в виде заголовка, а под ней — список коммитов от предыдущего релизного (или фиксрелизного) тега

3. Флоу фикса к релизу, который запускается вручную:

    - принимает на вход версию релиза и все действия выполняет в ветке этого релиза

    - запускает параллельно проверку типов и тесты

    - собирает docker-образ с двумя тегами:
        - `cr.yandex/<идентификатор_реестра>/app:<версия_релиза>_fix<номер_запуска_фиксрелизного_флоу>`
        - `cr.yandex/<идентификатор_реестра>/app:<версия_релиза>_latest`
    
    - загружает docker-образ в Container Registry (в реджистри должен появиться образ с двумя тегами, у старого образа тег latest пропадёт)

    - создаёт тег с номером текущего релиза + пометкой `fix` и номером фикса

    - добавляет комментарий в Issue, который содержит:
        - дату фикса,
        - автора фикса (тот, кто запустил флоу),
        - список коммитов от предыдущего релизного (или фиксрелизного) тега
        - ссылку на docker-образом в Yandex Container Registry\
          `cr.yandex/<идентификатор_реестра>/app:<версия_релиза>_fix<номер_запуска_фиксрелизного_флоу>`

4. Флоу выкатки релиза в прод

    - принимает на вход версию релиза

    - проверяет, что существует образ в Container Registry с тегом `<версия_релиза>_latest`

    - [по ssh запускает Docker-образ на виртуальной машине](https://yandex.cloud/ru/docs/container-registry/tutorials/run-docker-on-vm/console#run)

    - в Issue добавьте комментарий о том, что релиз выкачен в прод c датой и человеком, который запустил выкатку в прод

## Полезные ссылки

- [GitHub Actions](https://docs.github.com/ru/actions)
    - [Запуск NodeJS](https://docs.github.com/ru/actions/automating-builds-and-tests/building-and-testing-nodejs)
    - [Ручной запуск](https://docs.github.com/ru/actions/using-workflows/manually-running-a-workflow)
    - [Создать Issue через CLI](https://docs.github.com/ru/issues/tracking-your-work-with-issues/creating-an-issue#creating-an-issue-with-github-cli)

- [Docker](https://docs.docker.com/)
    - [Dockerfile](https://docs.docker.com/reference/dockerfile/)
    - [`docker init`](https://docs.docker.com/reference/cli/docker/init/)
    - [`docker build`](https://docs.docker.com/reference/cli/docker/image/build/)
    - [`docker run`](https://docs.docker.com/reference/cli/docker/container/run/)

## Запуск

```sh
# установите зависимости
npm ci

# соберите клиентский код приложения
npm run build

# запустите сервер
npm start
```

После этого можете открыть приложение в браузере по адресу http://localhost:3000/hw/store
