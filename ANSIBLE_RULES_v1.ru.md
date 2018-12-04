# Правила оформления Ansible-скриптов v1

Автор: Максим Лекомцев (https://github.com/abra7134)

## Общая структура
```shell
- bin/                  # Папка для хранения вспомогательных скриптов
- environments/         # Папка для хранения Ansible окружений
- roles/                # Папка для хранения Ansible ролей
- .gitignore            # Файл с исключениями для git
- requirements.txt      # Файл со списком необходимых python-зависимостей для работы с Ansible-скриптами
```

## Общие рекомендации
- в наименованиях любых файлов и папок допускается использовать только следующие символы: **abcdefghijklmnopqrstuvwxyz0123456789_.**
- любые комментарии в файлах указываются только на английском языке

### Структура roles/ папки
```shell
- roles/
  - имя_роли/
    - defaults/
      - main.yml
    - files/
      - файл_1
      - файл_2
      - ...
    - meta/
      - main.yml
    - tasks/
      - main.yml
      - имя_роли.yml
      - имя_роли_модуль_1.yml
      - имя_роли_модуль_2.yml
      - ...
    - templates/
      - темплейт_1.j2
      - темплейт_2.j2
      - ...
    - vars/
      - main.yml
    - README.md
  - ...
```

#### Рекомендации
- имя_роли должно локанично отображать суть того, что эта роль делает
- а краткое описание фиксируется в файле README.md, вот пример его наполнения:
  ```
  DOCKER_CONTAINERS
  =================

  A role for start Docker Registry with S3 backend in containers

  All needed parameters and his description see at defaults/main.yml
  ```

#### Содержимое
- `defaults/main.yml`:
  ```yaml
  # The AWS region in which your bucket exists
  docker_registry_s3_region: us-west-2
  # The bucket name in which you want to store the registry's data
  docker_registry_s3_bucket: ""

  # Authtentication method used by Docker Registry
  # Choice between:
  # * registry    - Use built-in in Docker Registry basic authentication
  #                 Allows specify only a list of users who have an access to Docker Registry
  # * docker_auth - Use Token+Basic authentication together with 'docker_auth' container
  #                 In addition allows specify a ACL list with grants of all specified users
  docker_registry_auth_by: registry

  # Dict with users and his password
  docker_registry_auth_users: {}
  #  admin: secret

  # Clean Docker Registry by follow methods:
  # * None          - Don't use a any cleaner
  # * registry-cli  - Using https://github.com/andrey-pohilko/registry-cli project
  #                   !!! This is work only if docker_registr_auth_by == registry
  docker_registry_cleaner_by: None
  ```
  * Здесь описываются все переменные, с помощью которых можно задавать поведение роли;
  * в `Ansible` переменные из этого источника носят низший уровень приоритета,
    т.е. могут быть переопределены во многих местах;
  * **Название** каждой переменной должно начинаться с **имени роли**, т.к. `Ansible` при своей работе
    помещает все используемые переменные в единое пространство имен. Тем самым при отладке будет
    сразу видно, где конкретная переменная будет использоваться;
  * Рекомендуется оформлять здесь емкие комментарии с описанием каждой переменной, а также использовать
    значения-заглушки на случаи, когда переменная нигде не переопределяется, т.е. фактически сразу
    задавать значения по-умолчанию;
  * Для сложных типов данных (таких как список или словарь) рекомендуется указывать пустые
    их значения с помощью `{}` или `[]` скобок. При необходимости указать пустое строкое значение -
    оформлять его в двойные `""` кавычки, в противном случае оно будет иметь `None` тип, а не `string`;
  * Для некоторых ситуация допускается использование `None` типа;
- `files/`:
  * Содержит статичные файлы для копирования через `Ansible`-модуль `copy`;
  * Рекомендуется повторять структуру папок для каждого файла, а именно того пути  куда файл фактически
    будет скопирован на целевой системе, например:
    ```
    - files/
      - opt/
        - bin/
          - script_1.sh
    ```
  * Если файл необходимо скопировать в несколько мест одновременно, рекомендуется использовать `symlink`
    относительно одного общего файла;
- `meta/main.yml`:
  ```yaml
  ---
  galaxy_info:
    author: Maksim Lekomtsev <lekomtsev@unix-mastery.ru>
    description: Role for running a Docker Registry with S3 backend for storage
    min_ansible_version: 2.7  ## TODO <- Need to minimize
    platforms:
      - name: Ubuntu
        versions:
          - bionic  ## TODO <- Add additional versions

  dependencies:
    - docker
  ```
  * Указывается общая информация для роли, а также ее зависимости;
  * Секция `galaxy_info` используется только при загрузки роли из [Ansible Galaxy](https://galaxy.ansible.com/),
    она избыточна, если разработка не публичная, однако в будущем даст возможность быстрой миграции
    в свою инсталяцию `Ansible Galaxy`. Вдобавок она стандартным способом задает информацию о том, на каких
    платформах эта роль будет работать;
- `tasks/`:
  * Содержит основную логику работы роли;
  * Более простой вариант содержит в себе файлы `main.yml` и `имя_роли.yml`:
    ```
    - tasks/
      - main.yml
      - docker_registry.yml
    ```
  * Более сложный вариант включает себя подмодули:
    ```
    - tasks/
      - main.yml
      - docker_registry.yml
      - docker_registry_docker_auth.yml
      - docker_registry_registry.yml
    ```
  * В обоих случаях `main.yml` является точкой входа и фактически используется только для указания тега для всей роли,
    чтобы дать возможность выборочного запуска составного `Ansible-playbook` (с использованием `-t` ключика):
    ```yaml
    ---
    - include_tasks: docker_registry.yml
      tags:
        - docker_registry
    ```
  * Первыми в роли рекомендуется помещать `Ansible`-задачи для проверки на соответствие типов переменных и значений:
    ```yaml
    - name: Checking variables types and values
      fail:
        msg: Invalid a variables type or value, please check
      when: >
        docker_registry_auth_by not in ['registry', 'docker_auth']     or
        docker_registry_auth_by == 'docker_auth'                       and
          (
            docker_registry_auth_token_acl is not sequence                 or
            docker_registry_auth_token_acl is string                       or
            docker_registry_auth_token_acl | count < 1                     or
            docker_registry_auth_token_certificate_pem is not string       or
            docker_registry_auth_token_certificate_pem == ""               or
            docker_registry_auth_token_key_pem is not string               or
            docker_registry_auth_token_key_pem == ""
          )
    ```
  * При офомрлении сложных параметров (как в пример выше) следует так разбивать их на несколько строк,
    чтобы максимально упростить чтение кода;
  * Рекомендуется использовать `YAML`-нотацию для указания параметров `Ansible`-модулей, избегать
    использование **строчной** нотации (хотя в некоторых случаях она может быть необходима), например:
    ```yaml
    - name: Checking variables types and values
      fail: msg="Invalid a variables type or value, please check"
    ```
  * **Имя** каждой задачи должно начинаться с глагола или существительного, написанного с заглавной буквы,
    и в кратце отображать суть того что происходит на конкретном шаге;
  * При использовании **подмодулей** рекомендуется в имя задачи добавлять имя подмодуля, а фактически
    имя файла, где это задача описана для более легкой отладки:
    ```yaml
    - name: "docker_auth : Create directories for configuration files"
      file:
        path: "{{ item }}"
        state: directory
      loop:
        - /opt/configs/docker_auth

    - name: "docker_auth : Write configuration files"
      template:
        src: "{{ item.strip('/') }}.j2"
        dest: "{{ item }}"
      loop:
        - /opt/configs/docker_auth/certificate.pem
        - /opt/configs/docker_auth/auth_config.yml
        - /opt/configs/docker_auth/key.pem
        - /opt/configs/registry/certificate.pem
    ```
  * При использовании `Ansible`-циклов (`loop`) преимущество нужно отдавать читабельности при выполнении `playbook`.
    В верхнем примере использование полных путей в списке `loop:` порождает необходимость использования
    `.strip('/')` функции, однако намного улучшает читабельность скрипта;
  * По возможности рекомендуется указывать параметры `Ansible`-модулей в **алфавитном** порядке:
    ```yaml
    - name: "docker_auth : Run 'docker_auth' container"
      docker_container:
        image: "{{ _docker_auth_image }}"
        name: docker_auth
        published_ports: "5001:5001"
        restart: yes
        restart_policy: always
        volumes:
          - "/opt/configs/docker_auth:{{ _docker_auth_config_dir }}:ro"
    ```
- `templates/`:
  * Содержит `Jinja2` файлы-шаблоны для копирования через `Ansible`-модуль `template`;
  * Рекомендуется повторять структуру папок для каждого файла, а именно пути куда он фактически
    будет скопирован на целевой системе, не забывая добавить расширение `.j2`:
    ```
    - templates/
      - etc/
        - cron.daily/
          - run_periodic_task.j2
    ```
  * Если файл-шаблон необходимо скопировать в несколько мест одновременно, рекомендуется использовать `symlink`
    относительно одного общего файла;
- `vars/main.yml`:
  * Здесь описываются **внутренние** переменные роли, поскольку в `Ansible` переменные из этого источника имеют высокий
    приоритет и не могут быть переопределены извне роли;
  * Название каждой переменной должно начинаться с символа подчеркивания `_` и имени роли, по аналогии
    с **private**-переменными в `Python`;
  * При необходимости рекомендуется в названиях переменных использовать уточняющие суффиксы-типы: `_list`, `_dict`, `_bool`, `_path`, `_dir`:
    ```yaml
    # Directories for configuration files inside containers
    _docker_registry_docker_auth_config_dir: /config
    _docker_registry_registry_config_dir:    /etc/docker/registry

    # Docker images names used in this Ansible's role
    _docker_registry_docker_auth_image:  "cesanta/docker_auth:1.3"
    _docker_registry_registry_image:     "registry:2"
    _docker_registry_registry_cli_image: "anoxis/registry-cli"
    ```
