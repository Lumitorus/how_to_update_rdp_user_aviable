```markdown
# Разрешить несколько одновременных RDP сеансов в Windows 10 и 11

Удаленные пользователи могут подключаться к своим компьютерам Windows 10 или 11 через службу удаленных рабочих столов (RDP). Достаточно включить удаленный рабочий стол (Remote Desktop), разрешить RDP доступ для пользователя [как описано здесь](https://winitpro.ru/index.php/2015/02/27/kak-razreshit-obychnym-polzovatelyam-rdp-dostup-k-kontrolleru-domena/) и подключиться к компьютеру с помощью любого клиента удаленного рабочего стола. Однако в десктопных версиях Windows есть ограничение на количество одновременных RDP сессий. Разрешается только один активный Remote Desktop сеанс пользователя.

Если вы попробуете открыть вторую RDP сессию, появится предупреждение с запросом отключить сеанс первого пользователя.

```
192.163.14.145 - Remote Desktop Connection

В систему вошел другой пользователь. Если вы продолжите, он будет отключен. Вы все равно хотите войти в систему?
```

В английской версии предупреждение такое:

```
Another user is signed in. If you continue, they'll be disconnected. Do you want to sign in anyway?
```

## Содержание:

- [Ограничения на количество RDP сессий в Windows](#ограничения-на-количество-rdp-сессий-в-windows)
- [RDP Wrapper: разрешить несколько RDP сеансов в Windows](#rdp-wrapper-разрешить-несколько-rdp-сеансов-в-windows)
- [Не работает RDP Wrapper в Windows](#не-работает-rdp-wrapper-в-windows)
- [Модификация файла termsrv.dll для снятия ограничений RDP в Windows 10 и 11](#модификация-файла-termsrvdll-для-снятия-ограничений-rdp-в-windows-10-и-11)
- [Встроенная поддержка нескольких RDP сессий в редакции Windows 10 Enterprise Multi-session](#встроенная-поддержка-нескольких-rdp-сессий-в-редакции-windows-10-enterprise-multi-session)

## Ограничения на количество RDP сессий в Windows

Во всех десктопных версиях Windows 10 и 11 есть ряд ограничений на использование служб удаленного рабочего стола:

1. Разрешено удаленно подключаться по RDP только к редакциям Windows Professional и Enterprise. В домашних редакциях (Home/Single Language) RDP доступ запрещен;
2. Поддерживается только одно одновременное RDP подключение. При попытке запустить вторую RDP-сессию, пользователю будет предложено завершить активный сеанс;
3. Если пользователь работает за консолью компьютера (локально), то при удаленном подключении по RDP, его локальный сеанс будет отключен (заблокирован). Также завершается и удаленный RDP сеанс, если пользователь входит в Windows через консоль компьютера.

Ограничение на количество одновременных RDP подключений в Windows является не техническим, но программным и лицензионным. Таким образом Microsoft запрещает создавать терминальный RDP сервер на базе рабочей станции для одновременной работы нескольких пользователей.

Если ваши задачи требуют развертывание терминального сервера, Microsoft предлагает приобрести Windows Server (по умолчанию разрешает 2 активных RDP подключения). Если вам нужно большее количество одновременных сессий пользователей, нужно приобрести лицензии RDS CAL [как описано здесь](https://winitpro.ru/index.php/2017/11/21/ustanovka-i-aktivaciya-rds-license-na-windows-server-2016/), установить и настроить роль Remote Desktop Session Host (RDSH) или полноценную RDS ферму [как описано здесь](https://winitpro.ru/index.php/2022/02/17/ustanovka-nastrojka-remote-desktop-services-rds-windows-server/).

Технически любая редакция Windows при наличии достаточного ресурса оперативной памяти и CPU может обслуживать одновременную работу нескольких десятков удаленных пользователей. В среднем на одну RDP сессию пользователя требуется 150-200 Мб памяти (без учета запускаемых приложений). Т.е. максимальное количество одновременных RDP сессий в теории ограничивается только ресурсами компьютера.

В этой статье мы покажем три способа убрать ограничение на количество одновременных RDP подключений в Windows 10 и 11:

- Использование RDP Wrapper
- Модификации системного файла termsrv.dll
- Апгрейд до редакции Windows 10/11 Enterprise for virtual desktops (multi-session)

**Примечание.** Все модификации операционной системы, описанные в этой статье, считаются нарушением лицензионного соглашения Windows, и вы можете использовать их на свой страх и риск.

Прежде, чем продолжить, проверьте, что в настройках Windows включен протокол Remote Desktop.

- Откройте панель **Settings -> System -> Remote Desktop** -> включите опцию **Enable Remote Desktop**;
- Либо воспользуйтесь классической панелью управления: выполните команду **SystemPropertiesRemote** -> Перейдите на вкладку **Remote Settings** (Удаленный доступ), включите опцию **Allow remote connection to this computer** (Разрешить удалённые подключения к этому компьютеру).

Более подробно о том, как включить и настроить RDP в Windows [в статье по ссылке](https://winitpro.ru/index.php/2013/04/29/kak-vklyuchit-udalennyj-rabochij-stol-v-windows-8/).

## RDP Wrapper: разрешить несколько RDP сеансов в Windows

Open-source утилита RDP Wrapper Library позволяет разрешить конкурентные RDP сессии в Windows 10/11 без замены системного файла termsrv.dll.

RDP Wrapper работает в качестве прослойки между менеджером управления службами Service Control Manager (SCM) и службой терминалов (Remote Desktop Services). RDP Wrapper не вносит никаких изменений в файл termsrv.dll, просто загружая termsrv с изменёнными параметрами.

**Важно.** Перед установкой RDP Wrapper важно убедиться, что у вас используется оригинальная (не пропатченная) версия файла termsrv.dll. Иначе RDP Wrapper может работать не стабильно, или вообще не запускаться.

Вы можете скачать RDP Wrapper из репозитория GitHub: [https://github.com/binarymaster/rdpwrap/releases](https://github.com/binarymaster/rdpwrap/releases) (последняя доступная версия RDP Wrapper Library v1.6.2). Утилита не обновляется с 2017 года, но ее можно использовать на всех билдах Windows 10 и 11. Для работы утилиты в современных версиях Windows достаточно обновить конфигурационный файл rdpwrap.ini.

Большинство антивирусов определяют RDP Wrapper как потенциально опасную программу. Например, встроенный Microsoft Defender антивирус классифицирует программу как PUA:Win32/RDPWrap (Potentially Unwanted Software) с низким уровнем угрозы. Если настройки вашего антивируса блокируют запуск RDP Wrapper, нужно добавить его в исключения.

Архив RDPWrap-v1.6.2.zip содержит несколько файлов:

- **RDPWinst.exe** - программа установки/удаления RDP Wrapper Library;
- **RDPConf.exe** - утилита настройки RDP Wrapper;
- **RDPCheck.exe** - Local RDP Checker - утилита для проверки RDP доступа;
- **install.bat, uninstall.bat, update.bat** - пакетные файлы для установки, удаления и обновления RDP Wrapper.

Чтобы установить RDPWrap, запустите файл **install.bat** с правами администратора.

После окончания установки запустите **RDPConfig.exe**.

Скорее всего, сразу после установки утилита покажет, что RDP wrapper запущен (Installed, Running, Listening), но не работает. Обратите внимание на красную надпись. Она сообщает, что данная версия Windows 10 (ver. 10.0.19041.1949) не поддерживается ([not supported]).

Причина в том, что в конфигурационном файле rdpwrap.ini отсутствует секция с настройками для вашей версии (билда) Windows. Актуальную версию файла rdpwrap.ini можно скачать здесь [https://raw.githubusercontent.com/sebaxakerhtc/rdpwrap.ini/master/rdpwrap.ini](https://raw.githubusercontent.com/sebaxakerhtc/rdpwrap.ini/master/rdpwrap.ini).

Вручную скопируйте содержимое данной страницы в файл **C:\Program Files\RDP Wrapper\rdpwrap.ini**. Или скачайте файл с помощью PowerShell командлета Invoke-WebRequest [как описано здесь](https://winitpro.ru/index.php/2014/10/08/obrabotka-soderzhimogo-veb-stranic-i-html-sajtov-v-powershell/) (предварительно нужно остановить службу Remote Desktop):

```powershell
Stop-Service termservice -Force
Invoke-WebRequest https://raw.githubusercontent.com/sebaxakerhtc/rdpwrap.ini/master/rdpwrap.ini -OutFile "C:\Program Files\RDP Wrapper\rdpwrap.ini"
```

**Примечание.** Можно создать задание планировщика для проверки изменений в файле rdpwrap.ini и его автоматического обновления.

На данном скриншоте видно, что на компьютере установлена свежая версия файла rdpwrap.ini (Updated=2023-06-26).

Перезагрузите компьютер, запустите утилиту **RDPConfig.exe**. Проверьте, что в секции Diagnostics все элементы окрашены в зеленый цвет, и появилось сообщение [Fully supported]. На скриншоте ниже показано, что RDP Wrapper с данным конфигом успешно запущен в Windows 11 22H2.

Теперь попробуйте установить несколько одновременных RDP сессий с этим компьютером под разными пользователями (воспользуйтесь любым RDP клиентом: mstsc.exe, RDCMan [как описано здесь](https://winitpro.ru/index.php/2011/01/12/upravlyaem-terminalnymi-podklyucheniyami-s-pomoshhyu-remote-desktop-connection-manager/), mRemoteNG и т.д.).

Можно использовать сохранённые RDP пароли [как описано здесь](https://winitpro.ru/index.php/2014/07/18/razreshaem-soxranenie-parolya-dlya-rdp-podklyucheniya/) для подключения к удаленному компьютеру.

Можете проверить, что на компьютере активны одновременно две RDP сессии (или более) с помощью команды:

```powershell
qwinsta
```

Пример вывода:

```
rdp-tcp#0 user1 1 Active
rdp-tcp#1 user2 2 Active
```

Утилита RDPWrap поддерживается во всех версиях Windows (включая домашние редакции Windows Home), таким образом из любой клиентской версии Windows можно сделать полноценный сервер терминалов.

В интерфейсе RDP Wrapper доступны следующие опции:

- **Enable Remote Desktop** - включить/отключить Remote Desktop в Windows доступ
- **RDP Port** - можно изменить стандартный номер порта удаленного рабочего стола TCP 3389 [как описано здесь](https://winitpro.ru/index.php/2010/09/17/nomer-porta-rdp-v-windows/)
- **Опция Hide users on logon screen** позволяет скрыть список пользователей на экране приветствия [как описано здесь](https://winitpro.ru/index.php/2014/10/16/windows-8-kak-skryt-polzovatelya-s-ekrana-privetstviya/);
- **Single session per user** - разрешить несколько одновременных RDP сессий под одной учетной записью пользователя. Эта опция устанавливает параметр реестра fSingleSessionPerUser = 0 в ветке HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server). Также этот параметр настраивается через опцию GPO Restrict Remote Desktop Services to a single Remote Desktop Services session в разделе Computer Configuration > Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Connections.
- В секции **Session Shadowing Mode** вы можете настроить режим теневого подключения к рабочему столу пользователей Windows [как описано здесь](https://winitpro.ru/index.php/2018/07/11/rdp-shadow-k-rabochemu-stolu-polzovatelya-windows-10/).

С помощью групповых политик вы можете настроить ограничения на длительность RDP сессий [как описано здесь](https://winitpro.ru/index.php/2020/05/25/rdp-session-limits/) пользователей в Windows. Это позволит автоматически отключать простаивающие сеансы.

## Не работает RDP Wrapper в Windows

В некоторых случаях утилита RDP Wrapper не работает как ожидается, и вы не можете использовать несколько RDP подключений.

Во время установки обновлений Windows может обновиться версия файла termsrv.dll. Если в файле rdpwrap.ini отсутствует описание для вашей версии Windows, значит RDP Wrapper не может применить необходимые настройки. В этом случае в окне RDP Wrapper Configuration будет указан статус [not supported].

В этом случае нужно обновить файл rdpwrap.ini как описано выше.

Если RDP Wrapper не работает после обновления файла rdpwrap.ini, попробуйте открыть файл rdpwrap.ini и найти в нем описание для вашей версии Windows.

Как понять, есть ли поддержка вашей версии Windows в конфиг файле rdpwrapper?

На скриншоте ниже показано, что для моей версии Windows 11 (10.0.22621.317) есть две секции с описаниями:

Если в конфигурационном файле rdpwrap соответствующая секция отсутствует для вашей версии Windows, попробуйте поискать в сети строки rdpwrap.ini для вашего билда. Добавьте найденные строки в самый конец файла.

Если после установки обновлений безопасности или после апгрейда билда Windows 10, RDP Wrapper не работает, проверьте, возможно в секции Diagnostics присутствует надпись Listener state: Not listening.

Попробуйте обновить ini файл, и затем переустановить службу:

```powershell
rdpwinst.exe -u
rdpwinst.exe -i
```

Бывает, что при попытке второго RDP подключения под другим пользователем у вас появляется надпись:

```
Число разрешенных подключений к этому компьютеру ограничено и все подключения уже используются. Попробуйте подключиться позже или обратитесь к системному администратору.
```

В этом случае нужно с помощью редактора групповых политик gpedit.msc [как описано здесь](https://winitpro.ru/index.php/2015/10/02/redaktor-gruppovyx-politik-dlya-windows-10-home-edition/) в секции Конфигурация компьютера -> Административные шаблоны -> Компоненты Windows -> Службы удаленных рабочих столов -> Узел сеансов удаленных рабочих столов -> Подключения включить политику "Ограничить количество подключений" и изменить ее значение на 999999 (Computer Configuration -> Administrative Templates -> Windows Components -> Remote Desktop Services -> Remote Desktop Session Host -> Connections -> Limit number of connections).

Перезагрузите компьютер для обновления настроек GPO [как описано здесь](https://winitpro.ru/index.php/2020/07/09/obnovlenie-gruppovyx-politik-windows/) и применения новых параметров.
```

Этот текст теперь отформатирован в формате Markdown, что упрощает его чтение и редактирование.
