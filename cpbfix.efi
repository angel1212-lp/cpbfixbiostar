/*
 * cpbfix.efi - clear (or set) the Core Performance Boost disable bit
 * (HWCR, MSR C001_0015, bit 25) on every logical processor, from the UEFI shell.
 *
 * Usage (from the UEFI shell):
 *     cpbfix.efi            clear the bit on all CPUs  -> boost allowed
 *     cpbfix.efi status     only read and report, change nothing
 *     cpbfix.efi disable    set the bit on all CPUs    -> boost off (undo)
 *
 * The change lasts until the machine is reset or powered off. It does not touch
 * the BIOS chip, UEFI variables, voltages or any power/temperature limit.
 *
 * No external headers: the few UEFI structures used are declared here.
 */

#ifdef HOSTTEST
#define EFIAPI __attribute__((ms_abi))
#else
#define EFIAPI
#endif

typedef unsigned char      UINT8;
typedef unsigned short     UINT16, CHAR16;
typedef unsigned int       UINT32;
typedef unsigned long long UINT64, UINTN;
typedef UINT8              BOOLEAN;
typedef void              *EFI_HANDLE, *EFI_EVENT;
typedef UINTN              EFI_STATUS;

#define EFI_SUCCESS       0ULL
#define EFI_ERR(x)        ((x) | 0x8000000000000000ULL)
#define EFI_UNSUPPORTED   EFI_ERR(3)
#define EFI_NOT_FOUND     EFI_ERR(14)
#define EFI_TIMEOUT       EFI_ERR(18)
#define HWCR_MSR          0xC0010015U
#define CPB_DIS_BIT       25
#define MAX_CPUS          512

typedef struct { UINT32 a; UINT16 b, c; UINT8 d[8]; } EFI_GUID;
typedef struct { UINT64 Signature; UINT32 Revision, HeaderSize, CRC32, Reserved; } EFI_TABLE_HEADER;

typedef struct EFI_MP_SERVICES EFI_MP_SERVICES;
typedef void (EFIAPI *EFI_AP_PROCEDURE)(void *Arg);
struct EFI_MP_SERVICES {
    EFI_STATUS (EFIAPI *GetNumberOfProcessors)(EFI_MP_SERVICES *, UINTN *Total, UINTN *Enabled);
    EFI_STATUS (EFIAPI *StartupAllAPs)(EFI_MP_SERVICES *, EFI_AP_PROCEDURE, BOOLEAN, EFI_EVENT, UINTN, void *, UINTN **);
    EFI_STATUS (EFIAPI *StartupThisAP)(EFI_MP_SERVICES *, EFI_AP_PROCEDURE, UINTN Cpu, EFI_EVENT, UINTN TimeoutUs, void *Arg, BOOLEAN *Finished);
    EFI_STATUS (EFIAPI *SwitchBSP)(EFI_MP_SERVICES *, UINTN, BOOLEAN);
    EFI_STATUS (EFIAPI *EnableDisableAP)(EFI_MP_SERVICES *, UINTN, BOOLEAN, UINT32 *);
    EFI_STATUS (EFIAPI *WhoAmI)(EFI_MP_SERVICES *, UINTN *Id);
};

typedef struct EFI_SIMPLE_TEXT_OUTPUT EFI_SIMPLE_TEXT_OUTPUT;
struct EFI_SIMPLE_TEXT_OUTPUT {
    void *Reset;
    EFI_STATUS (EFIAPI *OutputString)(EFI_SIMPLE_TEXT_OUTPUT *, CHAR16 *);
};

typedef struct {                       /* offsets must match UEFI spec order */
    EFI_TABLE_HEADER Hdr;
    void *RaiseTPL, *RestoreTPL, *AllocatePages, *FreePages, *GetMemoryMap,
         *AllocatePool, *FreePool, *CreateEvent, *SetTimer, *WaitForEvent,
         *SignalEvent, *CloseEvent, *CheckEvent, *InstallProtocolInterface,
         *ReinstallProtocolInterface, *UninstallProtocolInterface;
    EFI_STATUS (EFIAPI *HandleProtocol)(EFI_HANDLE, EFI_GUID *, void **);
    void *Reserved, *RegisterProtocolNotify, *LocateHandle, *LocateDevicePath,
         *InstallConfigurationTable, *LoadImage, *StartImage, *Exit, *UnloadImage,
         *ExitBootServices, *GetNextMonotonicCount, *Stall, *SetWatchdogTimer,
         *ConnectController, *DisconnectController, *OpenProtocol, *CloseProtocol,
         *OpenProtocolInformation, *ProtocolsPerHandle, *LocateHandleBuffer;
    EFI_STATUS (EFIAPI *LocateProtocol)(EFI_GUID *, void *Registration, void **Iface);
} EFI_BOOT_SERVICES;

typedef struct {
    EFI_TABLE_HEADER Hdr;
    CHAR16 *FirmwareVendor; UINT32 FirmwareRevision;
    EFI_HANDLE ConsoleInHandle; void *ConIn;
    EFI_HANDLE ConsoleOutHandle; EFI_SIMPLE_TEXT_OUTPUT *ConOut;
    EFI_HANDLE StandardErrorHandle; void *StdErr;
    void *RuntimeServices; EFI_BOOT_SERVICES *BootServices;
} EFI_SYSTEM_TABLE;

typedef struct { CHAR16 **Argv; UINTN Argc; } EFI_SHELL_PARAMETERS_PROTOCOL;

static EFI_GUID gMpGuid    = {0x3fdda605,0xa76e,0x4f46,{0xad,0x29,0x12,0xf4,0x53,0x1b,0x3d,0x08}};
static EFI_GUID gShellGuid = {0x752f3136,0x4e16,0x4fdc,{0xa2,0x2a,0xe5,0xf4,0x68,0x12,0xf4,0xca}};

/* ---- CPU access (replaced by fakes in the host test) ---- */
#ifdef HOSTTEST
extern UINT64 fake_rdmsr(UINT32 msr);
extern void   fake_wrmsr(UINT32 msr, UINT64 v);
extern void   fake_cpuid(UINT32 leaf, UINT32 *a, UINT32 *b, UINT32 *c, UINT32 *d);
#define RDMSR(m)       fake_rdmsr(m)
#define WRMSR(m, v)    fake_wrmsr(m, v)
#define CPUID(l,a,b,c,d) fake_cpuid(l,a,b,c,d)
#else
static inline UINT64 RDMSR(UINT32 m) { UINT32 lo, hi; __asm__ volatile("rdmsr" : "=a"(lo), "=d"(hi) : "c"(m)); return ((UINT64)hi << 32) | lo; }
static inline void WRMSR(UINT32 m, UINT64 v) { __asm__ volatile("wrmsr" :: "c"(m), "a"((UINT32)v), "d"((UINT32)(v >> 32))); }
static inline void CPUID(UINT32 l, UINT32 *a, UINT32 *b, UINT32 *c, UINT32 *d) { __asm__ volatile("cpuid" : "=a"(*a), "=b"(*b), "=c"(*c), "=d"(*d) : "a"(l), "c"(0)); }
#endif

/* ---- tiny output helpers ---- */
static EFI_SIMPLE_TEXT_OUTPUT *gOut;
static void puts16(const CHAR16 *s) { gOut->OutputString(gOut, (CHAR16 *)s); }
static void putdec(UINT64 v) {
    CHAR16 buf[24]; int i = 23; buf[i] = 0;
    do { buf[--i] = (CHAR16)(L'0' + v % 10); v /= 10; } while (v);
    puts16(&buf[i]);
}
static void puthex(UINT64 v) {
    CHAR16 buf[20]; int i = 19; buf[i] = 0;
    do { buf[--i] = (CHAR16)(L"0123456789ABCDEF"[v & 15]); v >>= 4; } while (v);
    puts16(L"0x"); puts16(&buf[i]);
}

/* ---- worker, runs on each core ---- */
typedef struct { UINT32 mode; UINT32 failed; UINT64 before, after; } CPU_RESULT;  /* mode 0=status 1=clear 2=set */
static CPU_RESULT gRes[MAX_CPUS];

static void EFIAPI Worker(void *arg) {
    CPU_RESULT *r = (CPU_RESULT *)arg;
    UINT64 v = RDMSR(HWCR_MSR);
    r->before = v;
    if (r->mode == 1) v &= ~(1ULL << CPB_DIS_BIT);
    else if (r->mode == 2) v |= (1ULL << CPB_DIS_BIT);
    if (r->mode != 0 && v != r->before) WRMSR(HWCR_MSR, v);
    r->after = RDMSR(HWCR_MSR);
}

static int lower(CHAR16 c) { return (c >= 'A' && c <= 'Z') ? c + 32 : c; }
static int streq(const CHAR16 *a, const char *b) {
    while (*a && *b) { if (lower(*a) != *b) return 0; a++; b++; }
    return *a == 0 && *b == 0;
}

EFI_STATUS EFIAPI efi_main(EFI_HANDLE image, EFI_SYSTEM_TABLE *st) {
    gOut = st->ConOut;
    EFI_BOOT_SERVICES *bs = st->BootServices;
    UINT32 mode = 1;

    /* arguments (shell only) */
    EFI_SHELL_PARAMETERS_PROTOCOL *sp = 0;
    if (bs->HandleProtocol(image, &gShellGuid, (void **)&sp) == EFI_SUCCESS && sp && sp->Argc >= 2) {
        CHAR16 *a = sp->Argv[1];
        if (streq(a, "status")) mode = 0;
        else if (streq(a, "disable")) mode = 2;
        else if (streq(a, "enable") || streq(a, "clear")) mode = 1;
        else { puts16(L"usage: cpbfix.efi [status|enable|disable]\r\n"); return EFI_UNSUPPORTED; }
    }

    puts16(L"cpbfix: HWCR[25] CPB-disable bit, all CPUs. Mode: ");
    puts16(mode == 0 ? L"status (read only)\r\n" : mode == 1 ? L"enable boost (clear bit)\r\n" : L"disable boost (set bit)\r\n");

    /* sanity: AMD, Zen or newer */
    UINT32 a, b, c, d;
    CPUID(0, &a, &b, &c, &d);
    if (!(b == 0x68747541 && d == 0x69746e65 && c == 0x444d4163)) { puts16(L"cpbfix: not an AMD CPU, refusing.\r\n"); return EFI_UNSUPPORTED; }
    CPUID(1, &a, &b, &c, &d);
    UINT32 fam = (a >> 8) & 0xF; if (fam == 0xF) fam += (a >> 20) & 0xFF;
    puts16(L"cpbfix: CPU family "); puthex(fam); puts16(L"\r\n");
    if (fam < 0x17) { puts16(L"cpbfix: family older than Zen, refusing.\r\n"); return EFI_UNSUPPORTED; }

    EFI_MP_SERVICES *mp = 0;
    if (bs->LocateProtocol(&gMpGuid, 0, (void **)&mp) != EFI_SUCCESS || !mp) {
        puts16(L"cpbfix: MP services protocol not found.\r\n"); return EFI_NOT_FOUND;
    }
    UINTN total = 0, enabled = 0, bsp = 0;
    mp->GetNumberOfProcessors(mp, &total, &enabled);
    mp->WhoAmI(mp, &bsp);
    puts16(L"cpbfix: processors "); putdec(total); puts16(L" (enabled "); putdec(enabled);
    puts16(L"), boot CPU is #"); putdec(bsp); puts16(L"\r\n");
    if (total == 0 || total > MAX_CPUS) { puts16(L"cpbfix: unexpected CPU count.\r\n"); return EFI_UNSUPPORTED; }

    UINTN okBefore = 0, okAfter = 0, done = 0;
    for (UINTN i = 0; i < total; i++) {
        gRes[i].mode = mode; gRes[i].failed = 0;
        if (i == bsp) {
            Worker(&gRes[i]);
        } else {
            BOOLEAN fin = 0;
            EFI_STATUS s = mp->StartupThisAP(mp, Worker, i, 0, 1000000, &gRes[i], &fin);
            if (s != EFI_SUCCESS) {
                gRes[i].failed = 1;
                puts16(L"  CPU "); putdec(i); puts16(L": could not run (status "); puthex(s); puts16(L") - skipped\r\n");
                continue;
            }
        }
        done++;
        UINT32 b0 = (UINT32)((gRes[i].before >> CPB_DIS_BIT) & 1), b1 = (UINT32)((gRes[i].after >> CPB_DIS_BIT) & 1);
        okBefore += b0; okAfter += b1;
        puts16(L"  CPU "); putdec(i); puts16(L": HWCR "); puthex(gRes[i].before);
        puts16(L" -> "); puthex(gRes[i].after);
        puts16(L"   bit25 "); putdec(b0); puts16(L" -> "); putdec(b1); puts16(L"\r\n");
    }
    puts16(L"cpbfix: cores reached "); putdec(done); puts16(L"/"); putdec(total);
    puts16(L"; CPB-disable bit set on "); putdec(okBefore); puts16(L" before, ");
    putdec(okAfter); puts16(L" after.\r\n");
    if (mode == 1 && okAfter == 0 && done == total) puts16(L"cpbfix: OK - boost is allowed on all CPUs until the next reset.\r\n");
    else if (mode == 1) puts16(L"cpbfix: WARNING - bit still set on some CPUs or some CPUs were skipped.\r\n");
    return EFI_SUCCESS;
}
