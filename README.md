# Pintos — Proje 1: Threads

Dokuz Eylül Üniversitesi **BİL 2008 İşletim Sistemleri** dersi Proje 1
(Threads) teslimi. Stanford Pintos eğitim çekirdeği üzerinde üç ana
işlevsellik baştan yazılmıştır:

1. **Alarm Clock** — busy-waiting'siz `timer_sleep`
2. **Priority Scheduling** — çoklu ve iç içe öncelik bağışı (priority donation)
3. **Advanced Scheduler** — 4.4BSD çok-seviyeli geri besleme kuyruğu (MLFQS)

## Test Durumu

`pintos/src/threads/build` altında `make check` (qemu):

```
All 27 tests passed.
```

Geçen testler: alarm-{single,multiple,simultaneous,priority,zero,negative},
priority-{change,donate-one,donate-multiple,donate-multiple2,donate-nest,
donate-sema,donate-lower,fifo,preempt,sema,condvar,donate-chain},
mlfqs-{load-1,load-60,load-avg,recent-1,fair-2,fair-20,nice-2,nice-10,block}.

## Derleme ve Test

Dev container (Ubuntu 22.04 + qemu + gdb) kullanılıyor; `.devcontainer/`
post-create script'i şunları otomatik yapıyor:

```bash
cd pintos/src/utils && make && sudo cp pintos Pintos.pm /usr/local/bin/
cd pintos/src/threads && make
```

Manuel derleme:

```bash
cd pintos/src/threads
make            # build/ altında kernel.bin, kernel.o üretir
make check      # 27 testi qemu üzerinde koşturur
make clean      # build/'i temizler
```

Tek bir test koşturmak için:

```bash
cd pintos/src/threads/build
pintos -- run alarm-multiple
pintos -- -mlfqs run mlfqs-fair-2     # MLFQS testleri için -mlfqs şart
```

### Build Notu (loader.bin)

Modern `binutils` (≥ 2.39) `loader.S`'i derlerken çıktıya
`.note.gnu.property` section'ı (40 byte) ekliyor; `ld --oformat binary`
de bunu dahil edip `loader.bin`'i 552 byte yapıyor. Pintos'un boot
yükleyici doğrulaması 314 veya 512 byte bekliyor. Düzeltme:
`pintos/src/Makefile.build` içindeki kuralın sonuna
`truncate -s 512 $@` eklendi. Boot sector imzası (`55 AA`) zaten 510.
offsette duruyor, kesme güvenli.

## Mimari Özet

### 1. Alarm Clock (`devices/timer.c` + `threads/thread.c`)

Eski `timer_sleep` `thread_yield`-döngüsü yapıyordu. Yeni akış:

- `timer_sleep(ticks)` kesmeleri kapatır, çağıran thread'i
  `sleeping_list`'e ekler ve `thread_block()` ile bloklar.
- Her timer kesmesinde `thread_tick()` `sleeping_list`'i tarar, her
  uyuyan thread'in `remaining_time_to_wake_up` sayacını 1 azaltır.
  Sayaç 0'a düştüğünde thread `thread_unblock` ile `ready_list`'e
  geri konur.
- Heap tahsisi yok; uyku metaverisi `struct thread` içinde gömülü.

Eklenen `struct thread` alanları:

```c
struct list_elem sleepelem;
int64_t remaining_time_to_wake_up;
```

### 2. Priority Scheduling (`threads/thread.c` + `threads/synch.c`)

Öncelik aralığı: `PRI_MIN=0`, `PRI_DEFAULT=31`, `PRI_MAX=63`.

**Hazır kuyruğu**: `ready_list` `list_insert_ordered` ile sıralı,
`list_pop_back` ile en yüksek öncelik alınır.

**Sema/Lock/Condvar**: 

- Sema `waiters`: sıralı tutulmaz; `sema_up` sırasında `list_max`
  ile O(n) tarama yapılır (donation sonrası önceliklerin değişebilmesi
  yüzünden bu yol seçildi).
- Condvar: `semaphore_elem.priority` alanı + `list_insert_ordered` +
  `list_pop_back`.

**Priority Donation (yalnız lock'lar için)**:

Eklenen alanlar:
- `struct thread`: `real_priority`, `locks_held` (liste),
  `current_lock` (bağ kurulan kilit)
- `struct lock`: `max_priority` (kilidin bekleyenlerinin en yükseği),
  `elem` (`locks_held`'e takılmak için)

Bir thread'in **etkili önceliği**:

```
priority = max( real_priority,
                max over l in locks_held of l.max_priority )
```

**İç içe bağış** (`lock_acquire` içinde): `current_lock` zinciri yukarı
takip edilerek her kilidin `max_priority`'si ve sahibinin etkili
önceliği güncellenir. Döngü, zaten yeterince yüksek bir bağışa
ulaşıldığında veya zincir kırıldığında (sahibin `current_lock == NULL`)
sonlanır.

**Bağışın geri alınması** (`lock_release`): kilit `locks_held`'den
çıkarılır ve `thread_update_priority` yeniden hesaplanır — diğer
kilitlerden gelen multiple donation korunur, yalnız bu kilitten gelen
bağış kalkar.

`thread_set_priority` `real_priority`'yi yazar, etkili önceliği
yeniden hesaplar, ardından `try_thread_yield` çağırır (daha öncelikli
biri varsa CPU bırakılır).

### 3. Advanced Scheduler — 4.4BSD MLFQS (`threads/thread.c` + `threads/fixed_point.h`)

`-mlfqs` çekirdek seçeneği ile aktif olur (`thread_mlfqs == true`).
Etkinken `thread_set_priority`/`thread_create`'in priority argümanı
yok sayılır.

**Sabit-nokta aritmetiği** (`fixed_point.h`): 17.15 formatı,
`fixed_t = int32_t`, kesir için en düşük 15 bit. Makrolar:
`FP_CONST`, `FP_ADD`, `FP_ADD_MIX`, `FP_SUB`, `FP_MULT`,
`FP_MULT_MIX`, `FP_DIV`, `FP_DIV_MIX`, `FP_INT_PART`, `FP_ROUND`.
`FP_MULT` ve `FP_DIV` taşma riski için içten `int64_t`'a yükseltir.

Eklenen alanlar:
- `struct thread`: `int nice`, `fixed_t recent_cpu`
- `thread.c`: `static fixed_t load_avg`

**Güncelleme zamanlaması** (`timer_interrupt` → `thread_tick`):

| Sıklık | İş |
|---|---|
| Her tikte | Çalışan thread'in `recent_cpu++` |
| Her 4 tikte | Çalışan thread'in `priority = PRI_MAX - recent_cpu/4 - nice*2` |
| Her TIMER_FREQ (100) tikte | `load_avg` ve tüm thread'lerin `recent_cpu` yeniden hesaplanır |

Formüller:

```
recent_cpu(t+1) = (2*load_avg) / (2*load_avg + 1) * recent_cpu + nice
load_avg(t+1)  = (59/60) * load_avg + (1/60) * ready_threads
priority       = PRI_MAX - (recent_cpu/4) - (nice*2)    [clamp PRI_MIN..PRI_MAX]
```

`thread_get_load_avg` ve `thread_get_recent_cpu`, ödev gereği 100 ile
çarpılıp yuvarlanmış tamsayı döndürür.

## Senkronizasyon Yaklaşımı

- `sleeping_list` ve `ready_list` paylaşımları için **kesme devre
  dışı bırakma**. Lock kullanılamıyor çünkü timer kesme handler'ı bu
  alanları okur ve kesme bağlamında `lock_acquire` çağrılamaz
  (`ASSERT(!intr_context())`).
- Kritik bölgeler **mümkün olan en kısa** tutuldu: `timer_sleep`'te
  liste ekleme + `thread_block`, `thread_set_priority`'de alan
  güncellemesi + `thread_update_priority`.
- Senkronizasyon araçlarının (semafor/lock/condvar) kendi içleri
  zaten kesme kapatma ile korunuyor; ek lock katmanı eklenmedi.

## Dizin Yapısı

```
pintos-den/
├── pintos/src/
│   ├── threads/
│   │   ├── DESIGNDOC          # Ödev tasarım belgesi (A1-A6, B1-B7, C1-C6)
│   │   ├── thread.c / .h      # Scheduler + donation + MLFQS + sleeping_list
│   │   ├── synch.c / .h       # Lock/sema/condvar + donation zinciri
│   │   └── fixed_point.h      # 17.15 sabit-nokta makroları (yeni)
│   ├── devices/
│   │   └── timer.c            # timer_sleep, timer_interrupt
│   └── Makefile.build         # loader.bin truncate fix
├── BekzadKopelov/             # (gitignored) Ödev kaynak metni ve şablon
└── .devcontainer/             # Ubuntu 22.04 + qemu + gdb
```

## Kısıtlar (Ödev Gereği)

- ❌ Busy waiting yok (`thread_yield`-döngüsü).
- ❌ Çıplak kesme devre dışı bırakmayla genel senkronizasyon yok —
  yalnız kernel ↔ interrupt handler paylaşımında, en kısa bölgede.
- ❌ Debug `printf` yok (kalan `printf`'ler orijinal Pintos kodu).
- ✅ `TIMER_FREQ=100`, `TIME_SLICE=4`, `PRI_MIN/DEFAULT/MAX` sabitleri
  korundu.
- ✅ Yığın ~4 kB; büyük yerel dizi kullanılmadı.

## Öğrenci

- **Bekzad Kopelov** — `2020280100@ogr.deu.edu.tr`

## Lisans

Pintos eğitim çekirdeği Stanford CS140 tarafından dağıtılan haliyle
kalır (BSD-benzeri lisans, kaynak dosyaların başlık yorumlarına bakın).
Bu repo, ders teslimi amacıyla eklenen değişiklikleri içerir.
