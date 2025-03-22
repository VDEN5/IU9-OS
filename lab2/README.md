# Гайд на лабораторную работу номер 2 (2025)

## Введение

Перед началом работы важно, чтобы первая лабораторная была уже сделана, я буду часто на нее ссылаться далее. Дальше мы будем всё делать на основании результатов с первой лабораторной, правда эти шаги придется ещё раз повторить, но про это чуть позже.

## ReactOS

### Сборка ReactOS с драйвером
Для начала удаляем исходники реактоса с первой лабораторной, затем нужно поочередно выполнить команды по-новой всё так же в RosBE:
```bash
git clone https://github.com/reactos/reactos
```
```bash
cd reactos/drivers
```
Теперь уже добавляем драйвер. В `CMakeLists.txt` дописываем строку ` add_subdirectory(lab2)` Создаем внутри папки драйверов папку `lab2`, затем уже в ней добавляем 3 файла со следующим содержимым:
`CMakeLists.txt`
```txt
add_library(yourdrivername MODULE yourdrivername.c yourdrivername.rc)

set_module_type(yourdrivername kernelmodedriver)

add_importlibs(yourdrivername ntoskrnl hal)

add_cd_file(TARGET yourdrivername DESTINATION reactos/system32/drivers FOR all)
```

`yourdrivername.c`
```c
#include <ntddk.h>

#ifndef NDEBUG

#define NDEBUG

#endif

#include <debug.h>

VOID NTAPI DriverUnload(IN PDRIVER_OBJECT DriverObject);

NTSTATUS NTAPI DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath)

{

DPRINT1("Ivan Ivanov");

DriverObject->DriverUnload = DriverUnload;

return STATUS_SUCCESS;

}

VOID NTAPI DriverUnload(IN PDRIVER_OBJECT DriverObject)

{

DPRINT1("Ivan Ivanov 2nd");

}
```

`yourdrivername.rc`
```rc
#define REACTOS_VERSION_DLL

#define REACTOS_STR_FILE_DESCRIPTION "Driver info"

#define REACTOS_STR_INTERNAL_NAME "yourdrivername"

#define REACTOS_STR_ORIGINAL_FILENAME "yourdrivername.sys"

#include <reactos/version.rc>
```

Теперь после создания файлов для драйверов переходим к сборке, тут все по-страому, нужно перейти в корень исходников и повторюсь тут:
Конфигурация: если винда
```bash
./configure.cmd
```
если линукс
```bash
./configure.sh
```
Дальше всё стандартно для винды и убунту:
```bash
cd output-MinGW-i386
```
```bash
ninja bootcd
```

### Работа в виртуальной машине

Тут тоже всё так же, как и раньше (не забываем про порты с отладочными логами в настройках). Но в прошлой лабе я посоветовал забить на дальнейшую настройку, потому напишу тут, что дальше лишь нужно выбрать язык и тупо везде прокликать. Теперь мы работаем в виртуальной машине. Заходим в `cmd.exe` (командную строку), в ней пишем:
```bash
sc create yourdrivername binPath= C:\ReactOS\system32\drivers\yourdrivername.sys type= kernel
```
Теперь очень важно. В строке выше после знака равно есть пробел, пишите с точностью до всех символов, командная строка в реактосе не прощает даже пропущенный пробел. И под конец:
```bash
sc start yourdrivername
```
По идее в логах должна появится ваша фамилия, это лучше проверить через поиск. Если нет, но все предыдущие пункты выполнены, то перезагрузите машину и последней командой еще раз запустите.
И еще: у меня сама строка показалась лишь спустя небольшое время внизу, но через поиск можно было найти мою фамилию, просто строку в начале не показывало почему-то.
## NetBSD

### Создание файлов драйвера
Ну тут всё по-новой. Как в первой лабе настраиваем машину и систему. Затем так же получаем исходники, дублировать не буду, смотрите первую лабу по момент установки исходников (то есть включительно, ядро собирать не надо). Теперь поочередно выполняем:
```bash
cd /usr
```
```bash
mkdir obj
```
```bash
cd src
```
```bash
./build.sh -U tools
```
```bash
cd sys/dev
```
```bash
vi lab2.c
```
Тем самым мы создаем файл драйвера при помощи vim, всё про работу в нем тоже в первой лабе смотреть. В нашем новом файле надо писать следующее:
```c
#include <sys/module.h>

MODULE(MODULE_CLASS_MISC, lab2, NULL);

static int lab2_modcmd(modcmd_t cmd, void* arg) {

printf("Ivan Ivanov");

return 0;

}
```
Далее:
```bash
cd ../modules
```
```bash
mkdir lab2
```
```bash
cd lab2
```
```bash
vi Makefile
```
В нем пишем следующее:
```
.include "../Makefile.inc"

KMOD=lab2

.PATH: ${S}/dev

SRCS=lab2.c

.include <bsd.kmodule.mk>
```
Затем:
```bash
make
```
```bash
modload ./lab2.kmod
```
После последней команды должна высветится в консоль ваша фамилия
Замечание: все действия эти делать под рутом, либо через суперпользователя, посредством `su`
### Сборка ядра

Всё в сборке так же (считаю, что находитесь в корневой папке), продублирую с первой лабы:
```bash
cd /usr/src/sys/arch/amd64/conf 
```
```bash
cp GENERIC MYKERN && config MYKERN
```
```bash
cd ../compile/MYKERN && make depend && make 
```
```bash
mv /netbsd /netbsd.old && mv netbsd /
```
Теперь перезапускаем машину, загружаем модуль два раза:
```bash
cd /usr/src/sys/modules/lab2
```
```bash
modload ./lab2.kmod
```
```bash
modload ./lab2.kmod
```
Загрузка модуля нужна, чтобы исскуственно вызвать ошибку присутствия этого модуля, только тогда в логах и появится ваша фамилия.
