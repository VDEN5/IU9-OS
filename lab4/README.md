# Гайд на лабораторную работу номер 4 (2025)

## Введение

Перед началом работы важно, чтобы вторая лабораторная была уже сделана, я буду часто на нее ссылаться далее. Дальше мы будем всё делать на основании опыта со второй лабораторной. По факту нам нужно лишь заменить строку с выводом фамилии на вывод страниц. Но ниже я повторю вторую лабораторную, лишь вставив везде нужные коды.

## ReactOS

### Сборка ReactOS с драйвером
Для начала удаляем исходники реактоса со второй лабораторной, затем нужно поочередно выполнить команды по-новой всё так же в RosBE:
```bash
git clone https://github.com/reactos/reactos
```
```bash
cd reactos/drivers
```
Теперь уже добавляем драйвер. В `CMakeLists.txt` дописываем строку ` add_subdirectory(lab4)` Создаем внутри папки драйверов папку `lab4`, затем уже в ней добавляем 4 файла со следующим содержимым:
`CMakeLists.txt`
```txt
add_library(yourdrivername MODULE yourdrivername.c yourdrivername.rc)

set_module_type(yourdrivername kernelmodedriver)

add_importlibs(yourdrivername ntoskrnl hal)

add_cd_file(TARGET yourdrivername DESTINATION reactos/system32/drivers FOR all)

add_registry_inf(yourdrivername.inf)
```

`yourdrivername.c`
```c
#include <ntddk.h>
#ifndef NDEBUG
#define NDEBUG
#endif
#include <debug.h>
#include <ntifs.h>
#include <ndk/ntndk.h>
#include <windef.h>


DRIVER_UNLOAD DriverUnload;
VOID NTAPI DriverUnload(IN PDRIVER_OBJECT DriverObject)
{
    DPRINT1("------------------DRIVER-UNLOADED--------------------\n");
    IoDeleteDevice(DriverObject->DeviceObject);
}

NTSTATUS NTAPI DriverEntry(IN PDRIVER_OBJECT DriverObject,
            IN PUNICODE_STRING RegistryPath)
{
    DriverObject->DriverUnload = DriverUnload;
    MmPageEntireDriver(DriverEntry);


    PVOID Pages = NULL;
    PHARDWARE_PTE PTE_BASE = (PHARDWARE_PTE)0xc0000000;
    PHARDWARE_PTE pte;
    SIZE_T sizeReserve = PAGE_SIZE * 10; 
    SIZE_T sizeCommit = PAGE_SIZE * 5;
    int i;

    ZwAllocateVirtualMemory(NtCurrentProcess(), &Pages, 0, &sizeReserve, MEM_RESERVE, PAGE_READWRITE);
    DPRINT1("10 PAGES RESERVED\n");

    ZwAllocateVirtualMemory(NtCurrentProcess(), &Pages, 0 , &sizeCommit, MEM_COMMIT, PAGE_READWRITE);
    DPRINT1("5 PAGES COMMITED\n");


    for(i = 0; i < 5; i++) {
        *((PCHAR)Pages + 0x1000 * i) = i + 1;
    }


    for(i = 0; i < 5; i++) {
        pte = ((ULONG)Pages >> 12) + PTE_BASE + i;
        DPRINT1("Page: %d\n\
        Physical address: %X\n\
        Valid:            %d\n\
        Accessed:         %d\n\
        Dirty:            %d\n\
        \n", i + 1, pte->PageFrameNumber<<12, pte->Valid,
         pte->Accessed,
        pte->Dirty);
    }
    ZwFreeVirtualMemory(NtCurrentProcess(), &Pages, &sizeCommit, MEM_DECOMMIT);
    DPRINT1("MEMORY IS DECOMMITED\n");
    ZwFreeVirtualMemory(NtCurrentProcess(), &Pages, 0, MEM_RELEASE);
    DPRINT1("MEMORY IS RELEASED\n");

    return STATUS_SUCCESS;
}
```

`yourdrivername.rc`
```rc
#define REACTOS_VERSION_DLL
#define REACTOS_STR_FILE_DESCRIPTION  "lab4 description"
#define REACTOS_STR_INTERNAL_NAME     "yourdrivername"
#define REACTOS_STR_ORIGINAL_FILENAME "yourdrivername.sys"
#include <reactos/version.rc>
```

`yourdrivername.inf`
```inf
; lab4
[AddReg]
HKLM,"SYSTEM\CurrentControlSet\Services\yourdrivername","ErrorControl",0x00010001,0x00000000
HKLM,"SYSTEM\CurrentControlSet\Services\yourdrivername","Group",0x00000000,"Base"
HKLM,"SYSTEM\CurrentControlSet\Services\yourdrivername","ImagePath",0x00020000,"system32\drivers\yourdrivername.sys"
HKLM,"SYSTEM\CurrentControlSet\Services\yourdrivername","Start",0x00000001,0x00000003
HKLM,"SYSTEM\CurrentControlSet\Services\yourdrivername","Type",0x00010001,0x00000001
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
По идее в логах должен появиться результат, это лучше проверить через поиск. Если нет, но все предыдущие пункты выполнены, то перезагрузите машину и последней командой еще раз запустите.
И еще: у меня сама строка показалась лишь спустя небольшое время внизу, но через поиск можно было найти мою фамилию, просто строку в начале не показывало почему-то.
## NetBSD

Если у вас всё ок со второй лабой, то чисто так делаем и всё:
```bash
cd /usr/src/sys/dev
```
```bash
vi lab4.c
```
Пишем (как именно писал ранее):
```c
#include <sys/module.h>
#include <sys/kernel.h>
#include <uvm/uvm_extern.h>
#include <uvm/uvm_pmap.h>
#include <uvm/uvm.h>
#include <sys/systm.h>
#include <sys/param.h>
#include <sys/queue.h>

MODULE(MODULE_CLASS_MISC, lab4, NULL);

#define NUM_PAGES 10

static struct vm_page *pglist = NULL;

static void print_page_info(vaddr_t va, int page_num) {
    paddr_t pa = 0;
    bool valid = pmap_extract(pmap_kernel(), va, &pa);
    bool used = false;
    bool modified = false;
    
    if (valid) {
        struct vm_page *pg = PHYS_TO_VM_PAGE(pa);
        if (pg != NULL) {
            used = pmap_is_referenced(pg);
            modified = pmap_is_modified(pg);
        }
    }

    printf("Page - %d\n", page_num);
    printf("Valid - %s\n", valid ? "true" : "false");
    printf("Used - %s\n", used ? "true" : "false");
    printf("Modified - %s\n", modified ? "true" : "false");
    printf("Physical address - 0x%08lx\n\n", (unsigned long)pa);
}

static int lab4_modcmd(modcmd_t cmd, void *arg) {
    vaddr_t addr;
    vsize_t size = NUM_PAGES * PAGE_SIZE;

    if (cmd == MODULE_CMD_INIT) {
        addr = uvm_km_alloc(kernel_map, size, PAGE_SIZE, UVM_KMF_VAONLY);
        if (addr == 0) {
            printf("lab4: virtual memory allocation failed\n");
            return ENOMEM;
        }

        struct pglist mlist;
        TAILQ_INIT(&mlist);
        int r = uvm_pglistalloc(5 * PAGE_SIZE, 0, 0xffffffff, 0, 0, &mlist, 5, 0);
        if (r != 0) {
            uvm_km_free(kernel_map, addr, size, 0);
            printf("lab4: physical memory allocation failed\n");
            return ENOMEM;
        }

        pglist = TAILQ_FIRST(&mlist); 

        struct vm_page *pg = TAILQ_FIRST(&mlist);
        for (int i = 0; i < 5 && pg != NULL; i++) {
            paddr_t pa = VM_PAGE_TO_PHYS(pg);
            pmap_kenter_pa(addr + i * PAGE_SIZE, pa, VM_PROT_READ | VM_PROT_WRITE, 0);
            pg = TAILQ_NEXT(pg, pageq.queue);
        }

        pmap_update(pmap_kernel());

        printf("Before free:\n");
        for (int i = 0; i < 10; i++) {
            print_page_info(addr + i * PAGE_SIZE, i + 1);
        }

        pmap_kremove(addr, size);
        pmap_update(pmap_kernel());

        if (pglist)
            uvm_pglistfree(&mlist);

        uvm_km_free(kernel_map, addr, size, 0);

        return 0;
    }

    if (cmd == MODULE_CMD_FINI) {
        printf("lab4: module unloaded\n");
        return 0;
    }

    return ENOTTY;
}
```
```bash
cd ../modules
```
```bash
mkdir lab4
```
```bash
cd lab4
```
```bash
vi Makefile
```
В нем пишем следующее:
```
.include "../Makefile.inc"

KMOD=lab4

.PATH: ${S}/dev

SRCS=lab4.c

.include <bsd.kmodule.mk>
```
Затем:
```bash
make
```
```bash
modload ./lab4.kmod
```
```bash
modload ./lab4.kmod
```
