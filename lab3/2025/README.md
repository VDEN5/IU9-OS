# Гайд на лабораторную работу номер 3 (2025)

## Введение

Перед началом работы важно, чтобы вторая лабораторная была уже сделана, я буду часто на нее ссылаться далее. Дальше мы будем всё делать на основании опыта со второй лабораторной. По факту нам нужно лишь заменить строку с выводом фамилии на вывод процессов. Но ниже я повторю вторую лабораторную, лишь вставив везде нужные коды.

## ReactOS

### Сборка ReactOS с драйвером
Для начала удаляем исходники реактоса со второй лабораторной, затем нужно поочередно выполнить команды по-новой всё так же в RosBE:
```bash
git clone https://github.com/reactos/reactos
```
```bash
cd reactos/drivers
```
Теперь уже добавляем драйвер. В `CMakeLists.txt` дописываем строку ` add_subdirectory(lab3)` Создаем внутри папки драйверов папку `lab3`, затем уже в ней добавляем 3 файла со следующим содержимым:
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
#include <ntifs.h>
#include <ndk/exfuncs.h>
#include <ndk/ketypes.h>
#include <ntstrsafe.h>
#ifndef NDEBUG 
#define NDEBUG 
#endif 
#include <debug.h>
 
NTSTATUS
NTAPI
DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath) {
    ULONG buffsz = 0;
    NTSTATUS status = ZwQuerySystemInformation(SystemProcessInformation, NULL, 0, &buffsz);
    UNICODE_STRING imagename;

    if (status != STATUS_INFO_LENGTH_MISMATCH) {
        return status;
    }
    
    PVOID process;
    while (0 == 0) {
        process = ExAllocatePoolWithTag(PagedPool, buffsz, 'Proc');
        if (!process) {
            return STATUS_MEMORY_NOT_ALLOCATED;
        }

        ULONG buffSize = 0;
        status = ZwQuerySystemInformation(SystemProcessInformation, process, buffsz, &buffSize);

        if (buffSize != buffsz) {
            ExFreePool(process);
            buffsz = buffSize;
            continue;
        }

        if (NT_ERROR(status)) {
            ExFreePool(process);
            return status;
        }

        break;
    }

    PSYSTEM_PROCESS_INFORMATION curtpr = (PSYSTEM_PROCESS_INFORMATION)process;
    while (curtpr) {
        RtlInitUnicodeString(&imagename, curtpr->ImageName.Buffer);

        DPRINT1("Name: %wZ, PID: %d, Parent PID: %d\n", 
                    &imagename, 
                    (ULONG)curtpr->UniqueProcessId,
                    (ULONG)curtpr->InheritedFromUniqueProcessId);

        if (!curtpr->NextEntryOffset) {
            break;
        }
        
        curtpr = (PSYSTEM_PROCESS_INFORMATION)((PUCHAR)curtpr + curtpr->NextEntryOffset);
    }

    ExFreePool(process);

    return STATUS_SUCCESS; 
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

Теперь после создания файлов для драйверов переходим к сборке, тут все по-старому, нужно перейти в корень исходников и повторюсь тут:
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

Тут тоже всё так же, как и раньше (не забываем про порты с отладочными логами в настройках). Теперь мы работаем в виртуальной машине. Заходим в `cmd.exe` (командную строку), в ней пишем:
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

### Важное замечание, чтоб быстрее сделать.
Если у вас всё ок со второй лабой, то чисто так делаем и всё:
```bash
cd usr/src/sys/dev
```
```bash
vi lab3.c
```
Пишем (как именно писал ранее):
```c
#include <sys/module.h>

MODULE(MODULE_CLASS_MISC, lab3, NULL);

static int lab3_modcmd(modcmd_t cmd, void* arg) {
    struct proc *p;
    switch (cmd) {
        case MODULE_CMD_INIT:
            printf("Initializing lab3 driver\n");
            PROCLIST_FOREACH(p, &allproc) {
                printf("Process ID: %d, Name: %s\n", p->p_pid, p->p_comm);
            }
            break;
        case MODULE_CMD_FINI:
            printf("Shutting down lab3 driver\n");
            break;
        default:
            return ENOTTY;
    }
    return 0;
}
```
```bash
cd ../modules
```
```bash
mkdir lab3
```
```bash
cd lab3
```
```bash
vi Makefile
```
В нем пишем следующее:
```
.include "../Makefile.inc"

KMOD=lab3

.PATH: ${S}/dev

SRCS=lab3.c

.include <bsd.kmodule.mk>
```
Затем:
```bash
make
```
```bash
modload ./lab3.kmod
```
```bash
modload ./lab3.kmod
```

### Создание файлов драйвера
Ну тут всё по-новой. Как в первой лабе настраиваем машину и систему. Затем так же получаем исходники, дублировать не буду, смотрите первую лабу по момент установки исходников (то есть включительно, ядро собирать не надо). Теперь поочередно выполняем (но если сделали 2 лабу, то это можно скипнуть до момента создания кода):
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
vi lab3.c
```
Тем самым мы создаем файл драйвера при помощи vim, всё про работу в нем тоже в первой лабе смотреть. В нашем новом файле надо писать следующее:
```c
#include <sys/module.h>

MODULE(MODULE_CLASS_MISC, lab3, NULL);

static int lab3_modcmd(modcmd_t cmd, void* arg) {
    struct proc *p;
    switch (cmd) {
        case MODULE_CMD_INIT:
            printf("Initializing lab3 driver\n");
            PROCLIST_FOREACH(p, &allproc) {
                printf("Process ID: %d, Name: %s\n", p->p_pid, p->p_comm);
            }
            break;
        case MODULE_CMD_FINI:
            printf("Shutting down lab3 driver\n");
            break;
        default:
            return ENOTTY;
    }
    return 0;
}
```
Далее:
```bash
cd ../modules
```
```bash
mkdir lab3
```
```bash
cd lab3
```
```bash
vi Makefile
```
В нем пишем следующее:
```
.include "../Makefile.inc"

KMOD=lab3

.PATH: ${S}/dev

SRCS=lab3.c

.include <bsd.kmodule.mk>
```
Затем:
```bash
make
```
```bash
modload ./lab3.kmod
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
modload ./lab3.kmod
```
```bash
modload ./lab3.kmod
```
Загрузка модуля нужна, чтобы исскуственно вызвать ошибку присутствия этого модуля, только тогда в логах и появятся данные.
