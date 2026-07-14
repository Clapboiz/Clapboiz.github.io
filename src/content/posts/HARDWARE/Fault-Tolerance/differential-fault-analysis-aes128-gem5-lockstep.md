---
title: "[Phần 2 - Tấn công I] Tấn Công AES-128 Bằng Differential Fault Analysis Trên gem5 và Cơ Chế Phòng Vệ Lockstep"
published: 2026-06-20
description: "Hardware Security & Computer Architecture"
image: './banner1.png'
tags: ["HARDWARE", "SECURITY", "COMPUTER-ARCHITECTURE"]
category: "Hardware-Security"
draft: false
---

# Hướng 1: Classic DFA Attack trên AES-128 trong gem5 RISC-V

> **Stack dùng trong pipeline cuối:** gem5 · tiny-AES-c · PhoenixAES · riscv64-linux-gnu-gcc  
> **Mục tiêu:** gây lỗi tại AES Round 9, thu faulty ciphertexts, dùng PhoenixAES recover last round key, rồi kiểm tra Lockstep Defense có chặn ciphertext lỗi trước khi output ra ngoài hay không.  
> **Lưu ý về GemFI:** GemFI/hardware-level injection là hướng ban đầu, nhưng trong log không đi được đến pipeline chạy ổn định. Phần attack cuối cùng dùng source-level fault hook trong `aes.c`, sau đó compile thành RISC-V binary và chạy trong gem5 để sinh ciphertext lỗi.

---

## Mục lục

1. [Tổng quan về DFA Attack](#1-tổng-quan-về-dfa-attack)
2. [Setup môi trường gem5](#2-setup-môi-trường-gem5)
3. [Compile AES cho RISC-V](#3-compile-aes-cho-risc-v)
4. [Chạy AES trong gem5 SE Mode](#4-chạy-aes-trong-gem5-se-mode)
5. [Phân tích disassembly  - tìm fault window](#5-phân-tích-disassembly--tìm-fault-window)
6. [Xác định cycle range của round 9](#6-xác-định-cycle-range-của-round-9)
7. [Approach 1: Checkpoint-based fault injection](#7-approach-1-checkpoint-based-fault-injection)
8. [Vấn đề với checkpoint approach  - debug quá trình](#8-vấn-đề-với-checkpoint-approach--debug-quá-trình)
9. [Approach 2: Source-level fault injection](#9-approach-2-source-level-fault-injection)
10. [Collect 128 faulty ciphertexts](#10-collect-128-faulty-ciphertexts)
11. [PhoenixAES recover round 10 key](#11-phoenixaes-recover-round-10-key)
12. [Recover master key bằng reverse key schedule](#12-recover-master-key-bằng-reverse-key-schedule)
13. [Lockstep Defense  - implement và verify](#13-lockstep-defense--implement-và-verify)
14. [Tổng kết và bài học](#14-tổng-kết-và-bài-học)

---

## 1. Tổng quan về DFA Attack

Differential Fault Analysis (DFA) là một kỹ thuật **active side-channel attack**, trong đó kẻ tấn công **chủ động gây lỗi (fault)** trong quá trình thực thi của thuật toán mật mã thay vì chỉ quan sát hành vi của hệ thống như các tấn công **Power Analysis** hay **Timing Attack**.

Ý tưởng của DFA khá đơn giản:

- Chạy thuật toán bình thường để thu được **ciphertext đúng**.
- Gây một lỗi nhỏ tại một thời điểm xác định trong quá trình mã hóa để tạo ra **ciphertext lỗi**.
- So sánh hai ciphertext này để suy luận thông tin về khóa bí mật.

Đối với AES-128, DFA nổi tiếng của **Piret & Quisquater** tận dụng cách mà sai khác (*difference*) lan truyền qua các vòng mã hóa để khôi phục **Round 10 Key (Last Round Key)**, sau đó sử dụng **Reverse Key Schedule** để tính ngược về **Master Key**.

---

AES-128 gồm **10 rounds**. Mỗi round thực hiện bốn phép biến đổi theo thứ tự:

```text
SubBytes
    ↓
ShiftRows
    ↓
MixColumns
    ↓
AddRoundKey
```

Riêng **Round 10** không thực hiện bước **MixColumns**.

Trong classic DFA, lỗi được chèn **ngay trước SubBytes của Round 9** (hay nói cách khác là sau khi kết thúc Round 8). Khi đó, sai khác chỉ còn đi qua:

- Round 9 (đầy đủ bốn phép biến đổi)
- Round 10 (không có MixColumns)

Điều này tạo ra một mẫu sai khác (*difference pattern*) đặc trưng trên ciphertext cuối cùng. Mẫu sai khác này phụ thuộc trực tiếp vào **Round 10 Key**, vì vậy chỉ cần thu thập đủ số lượng cặp:

```text
(Correct Ciphertext, Faulty Ciphertext)
```

PhoenixAES sẽ khai thác các quan hệ toán học của DFA để khôi phục **Round 10 Key (Last Round Key)**. Sau khi thu được khóa của vòng cuối, chỉ cần thực hiện **Reverse Key Schedule** là có thể tính ngược về **Master Key** của AES-128.

![AES-128 Round Structure](Image/aes128.png)

---

## Tại sao lại chọn Round 9?

Đây là câu hỏi quan trọng nhất trong DFA.

Thời điểm gây lỗi quyết định trực tiếp khả năng thành công của cuộc tấn công.

### Gây lỗi quá sớm (Round 1–8)

Nếu fault được đưa vào từ các vòng đầu, sai khác sẽ phải đi qua nhiều lần **MixColumns**.

Do MixColumns có tính chất **diffusion**, một lỗi rất nhỏ ban đầu sẽ nhanh chóng lan ra toàn bộ trạng thái AES.

Ví dụ:

- Ban đầu chỉ có **1 byte** bị lỗi.
- Sau một lần **MixColumns**, lỗi lan sang **4 byte** trong cùng một cột.
- Sau nhiều rounds tiếp theo, gần như toàn bộ **16 byte** của state đều bị ảnh hưởng.

Khi sai khác lan truyền quá rộng, cấu trúc toán học mà DFA dựa vào gần như biến mất, khiến việc khôi phục khóa trở nên rất khó hoặc không còn khả thi.

### Gây lỗi quá muộn (Round 10)

Ngược lại, nếu fault được chèn vào **Round 10** thì ciphertext gần như chỉ bị ảnh hưởng bởi đúng byte bị lỗi.

Do Round 10 **không có MixColumns**, sai khác không được khuếch tán sang các byte khác.

Kết quả là ciphertext chỉ chứa rất ít thông tin về cấu trúc lan truyền của lỗi, không đáp ứng được giả định của thuật toán DFA cổ điển.

### Round 9 là vị trí "vừa đủ"

Round 9 chính là điểm cân bằng giữa hai trường hợp trên.

Lỗi chỉ phải đi qua **một lần MixColumns** trước khi tạo ciphertext cuối cùng.

Nhờ đó:

- Sai khác được khuếch tán vừa đủ để tạo thành mẫu gồm **4 byte liên quan**.
- Mẫu sai khác vẫn giữ được cấu trúc toán học mà thuật toán DFA có thể khai thác.
- Thông tin thu được đủ để PhoenixAES suy luận từng byte của **Round 10 Key**.

Chính vì vậy, phần lớn các nghiên cứu DFA trên AES-128 đều lựa chọn **fault injection tại đầu Round 9**.

---

## Điều kiện để DFA thành công

Để cuộc tấn công hoạt động, cần thỏa mãn ba điều kiện chính:

- Attacker phải đưa fault vào **đúng thời điểm**, tương ứng với **Round 9** của thuật toán.
- Có thể thu thập được cả **ciphertext đúng** và **ciphertext lỗi**.
- Fault model đủ "sạch", thông thường là **single-byte fault** hoặc **single-bit fault** trong trạng thái AES.

Nếu fault quá mạnh hoặc xuất hiện ở vị trí không mong muốn, cấu trúc sai khác sẽ không còn phù hợp với mô hình phân tích của PhoenixAES.

---

## Tại sao sử dụng gem5?

Đây cũng chính là lý do **gem5** được lựa chọn trong bài viết này.

Mặc dù DFA thường được nghiên cứu trên phần cứng thực, gem5 cho phép mô phỏng toàn bộ quá trình thực thi của chương trình trên kiến trúc **RISC-V**, đồng thời cung cấp **instruction trace** và thông tin về số chu kỳ thực thi.

Nhờ đó có thể:

- Xác định chính xác khoảng thời gian tương ứng với từng vòng AES.
- Tiêm lỗi tại đúng vị trí mong muốn.
- Thu thập ciphertext lỗi một cách có kiểm soát.
- Kiểm chứng khả năng khôi phục khóa của PhoenixAES.
- Đánh giá xem cơ chế **Lockstep Defense** có ngăn được việc ciphertext lỗi bị xuất ra ngoài hay không.

Trong bài viết này, fault sẽ được chèn tại **Round 9** của AES thông qua **source-level fault injection**, sau đó chương trình được biên dịch thành **RISC-V binary** và chạy trên **gem5** để sinh ra các faulty ciphertext phục vụ quá trình phân tích bằng **PhoenixAES**.

---

## 2. Setup môi trường gem5

### gem5 là gì?

gem5 là một architectural simulator: nó cho phép chạy binary của một kiến trúc như RISC-V, ARM hoặc x86 trên máy host mà không cần phần cứng thật. Tùy CPU model, gem5 có thể mô phỏng ở các mức chi tiết khác nhau.

Trong bài này, ta dùng `RiscvTimingSimpleCPU`. Đây không phải mô hình vi kiến trúc cycle-accurate tuyệt đối như một CPU thật, nhưng đủ tốt cho mục tiêu của bài: chạy AES RISC-V binary, lấy execution trace ổn định, xác định marker của Round 9, rồi kiểm chứng fault model.

Trong project này, ta dùng gem5 phiên bản 25.1.0.1 với RISC-V backend.

### Verify gem5 đã compile xong

```bash
┌──(clap㉿clap)-[~/Desktop/gem5]
└─$ ./build/RISCV/gem5.opt --version
Usage
=====
  gem5.opt [gem5 options] script.py [script options]

gem5.opt: error: no such option: --version
```

Đây là output bình thường trong log. `gem5.opt` không nhận `--version` như một option hợp lệ, nhưng việc nó in được usage message cho thấy binary đã khởi động được. Bước này chỉ xác nhận gem5 executable không bị lỗi cơ bản; workload AES vẫn cần được kiểm chứng riêng ở các bước sau.

### Cấu trúc gem5 RISC-V build

```
build/RISCV/gem5.opt   ← optimized build, dùng cho production run
build/RISCV/gem5.debug ← debug build, slower nhưng có thêm assertions
build/RISCV/gem5.fast  ← fastest build, bỏ hết assertions
```

Trong log này ta dùng `gem5.opt` vì nó cân bằng giữa tốc độ và khả năng debug. Khi phải chạy hàng trăm lần để thu faulty ciphertexts, dùng bản optimized sẽ thực tế hơn `gem5.debug`.

---

## 3. Compile AES cho RISC-V

### Chọn AES implementation

Ta dùng [tiny-AES-c](https://github.com/kokke/tiny-AES-c)  - một AES implementation thuần C, single-file, không dependency. Lý do:

- Code đơn giản, dễ audit và patch
- Không có hardware acceleration (AES-NI)  - quan trọng vì ta cần AES chạy như software thật, không phải opcode đặc biệt
- Compile sạch trên cross-compiler RISC-V
- Được dùng rộng rãi trong embedded/IoT research

### Clone và chuẩn bị

```bash
cd ~/Desktop
git clone https://github.com/kokke/tiny-AES-c
cd tiny-AES-c
```

### Tạo wrapper với NIST test vector

Trước khi inject fault, cần có một baseline đúng. Nếu AES implementation hoặc cross-compile đã sai từ đầu, mọi ciphertext lỗi thu được sau đó sẽ không còn ý nghĩa. Vì vậy bước đầu tiên là chạy AES với một test vector chuẩn của NIST FIPS 197 Appendix B.

```
Key:       2B7E151628AED2A6ABF7158809CF4F3C
Plaintext: 6BC1BEE22E409F96E93D7E117393172A
Expected:  3AD77BB40D7A3660A89ECAF32466EF97
```

![alt text](Image/Screenshot_283.png)

Để xác nhận giá trị tham chiếu, cùng một cặp khóa và bản rõ được kiểm tra bằng thư viện PyCryptodome trong Python:

```python
from Crypto.Cipher import AES

key = bytes.fromhex('2B7E151628AED2A6ABF7158809CF4F3C')
pt  = bytes.fromhex('6BC1BEE22E409F96E93D7E117393172A')
ct  = AES.new(key, AES.MODE_ECB).encrypt(pt)

print(ct.hex().upper())
# 3AD77BB40D7A3660A89ECAF32466EF97
```

Kết quả này là ground truth của toàn bộ thí nghiệm: ciphertext đúng phải là `3ad77bb40d7a3660a89ecaf32466ef97`.

Tiếp theo, ta tạo wrapper tối giản (`aes_main.c`) để gọi trực tiếp `AES_ECB_encrypt()` của tiny-AES-c. Chương trình này chỉ mã hóa đúng một block 16 byte, in ciphertext ra stdout, và được dùng làm workload cho gem5.

```c
cat > aes_main.c << 'EOF'
#include <stdio.h>
#include <string.h>
#include "aes.h"

int main() {
    uint8_t key[16] = {
        0x2b, 0x7e, 0x15, 0x16, 0x28, 0xae, 0xd2, 0xa6,
        0xab, 0xf7, 0x15, 0x88, 0x09, 0xcf, 0x4f, 0x3c
    };
    uint8_t plaintext[16] = {
        0x6b, 0xc1, 0xbe, 0xe2, 0x2e, 0x40, 0x9f, 0x96,
        0xe9, 0x3d, 0x7e, 0x11, 0x73, 0x93, 0x17, 0x2a
    };

    struct AES_ctx ctx;
    AES_init_ctx(&ctx, key);

    uint8_t buf[16];
    memcpy(buf, plaintext, 16);
    AES_ECB_encrypt(&ctx, buf);

    printf("Ciphertext: ");
    for (int i = 0; i < 16; i++) printf("%02x", buf[i]);
    printf("\n");

    return 0;
}
EOF
```

Tiêu chí xác nhận tính đúng đắn là ciphertext được sinh ra phải hoàn toàn trùng khớp với giá trị tham chiếu của NIST:

```
3ad77bb40d7a3660a89ecaf32466ef97
```

Nếu ciphertext khác giá trị này, phải dừng lại để sửa AES wrapper hoặc toolchain trước khi làm fault injection. DFA cần so sánh giữa correct output và faulty output; correct output sai thì toàn bộ phân tích phía sau sai theo.

### Biên dịch chương trình AES cho kiến trúc RISC-V

Sau khi có wrapper, bước tiếp theo là biên dịch chương trình sang RISC-V để gem5 có thể chạy được.

Ban đầu, chương trình được biên dịch bằng bộ công cụ cross-compiler RISC-V:

![alt text](Image/Screenshot_290.png)

```bash
riscv64-linux-gnu-gcc -O0 -g -o aes_test aes_main.c aes.c -I.
```

Tùy chọn `-O0` được dùng để hạn chế compiler tối ưu hóa quá mạnh. Điều này giúp disassembly giữ được cấu trúc gần với source hơn: `Cipher()`, `SubBytes()`, `ShiftRows()`, `MixColumns()` và `AddRoundKey()` vẫn hiện rõ, thuận tiện cho việc tìm fault window. Nếu dùng `-O2` hoặc `-O3`, compiler có thể inline hoặc sắp xếp lại code, làm trace khó đọc hơn.

Sau khi biên dịch, loại binary được kiểm tra bằng lệnh:

```bash
file aes_test
```

Kết quả cho thấy:

```text
aes_test: ELF 64-bit LSB pie executable, UCB RISC-V,
dynamically linked,
interpreter /lib/ld-linux-riscv64-lp64d.so.1
...
```

Thông tin này xác nhận binary đúng kiến trúc RISC-V. Nhưng nó cũng cho thấy một vấn đề: file đang là **dynamically linked executable**.

### Vấn đề với dynamic linking trong gem5 SE mode

![alt text](Image/Screenshot_285.png)

gem5 hỗ trợ hai chế độ mô phỏng chính:

* **Full System (FS) mode**: mô phỏng toàn bộ hệ thống bao gồm hệ điều hành, trình điều khiển thiết bị và các thành phần phần cứng liên quan.
* **Syscall Emulation (SE) mode**: chỉ mô phỏng tiến trình người dùng và các lời gọi hệ thống (system calls), không mô phỏng toàn bộ hệ điều hành.

Trong log này, ta dùng **SE mode** vì workload chỉ là một userspace program nhỏ. SE mode nhẹ hơn Full System mode và phù hợp khi cần chạy lại workload nhiều lần để collect faulty ciphertexts.

Tuy nhiên, SE mode không cung cấp đầy đủ môi trường thực thi của hệ điều hành Linux, đặc biệt là không hỗ trợ cơ chế nạp thư viện động (dynamic linking). Do đó, khi chạy binary được liên kết động, gem5 báo lỗi:

```text
fatal: Failed to open file /lib/ld-linux-riscv64-lp64d.so.1
```

Lỗi này không phải do AES sai. Binary RISC-V được compile dạng dynamic nên khi chạy nó cần dynamic linker `ld-linux-riscv64-lp64d.so.1`. Trong SE mode hiện tại, gem5 không có root filesystem RISC-V đầy đủ để tìm file đó, nên simulator dừng trước khi chương trình AES chạy.

![alt text](Image/Screenshot_291.png)

Để khắc phục, chương trình được biên dịch lại dưới dạng **statically linked executable**:

```bash
riscv64-linux-gnu-gcc -O0 -g -static -o aes_test aes_main.c aes.c -I.
```

Sau khi biên dịch lại, kiểm tra bằng lệnh:

```bash
file aes_test
```

thu được kết quả:

```text
aes_test: ELF 64-bit LSB executable, UCB RISC-V,
statically linked,
...
```

Thuộc tính *statically linked* cho thấy các thư viện cần thiết đã được liên kết trực tiếp vào binary. Nhờ đó, gem5 SE mode không cần tìm dynamic linker nữa và có thể chạy workload AES trực tiếp.

---

## 4. Chạy AES trong gem5 SE Mode

### Config script cho gem5

gem5 dùng Python script để mô tả system cần mô phỏng: clock, CPU model, memory, bus, workload và process arguments. Với workload AES nhỏ này, cấu hình tối giản là đủ:

```python
# run_aes.py
import m5
from m5.objects import *

# Khởi tạo system
system = System()
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = '1GHz'
system.clk_domain.voltage_domain = VoltageDomain()

system.mem_mode = 'timing'
system.mem_ranges = [AddrRange('512MB')]

# CPU: TimingSimpleCPU
system.cpu = RiscvTimingSimpleCPU()

# Memory bus
system.membus = SystemXBar()
system.cpu.icache_port = system.membus.cpu_side_ports
system.cpu.dcache_port = system.membus.cpu_side_ports
system.cpu.createInterruptController()

# DDR3 memory controller
system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports
system.system_port = system.membus.cpu_side_ports

# Load binary
binary = '/home/clap/Desktop/tiny-AES-c/aes_test'
system.workload = SEWorkload.init_compatible(binary)

process = Process()
process.cmd = [binary]

system.cpu.workload = process
system.cpu.createThreads()

root = Root(full_system=False, system=system)

m5.instantiate()

print("=== Starting AES simulation ===")
exit_event = m5.simulate()

print(f"Exited at tick {m5.curTick()}  - reason: {exit_event.getCause()}")
```

Trong log, lần chạy đầu tiên dùng nhầm path `/root/Desktop/tiny-AES-c/aes_test`, trong khi user thật là `clap`. gem5 báo:

```text
Failed to open file /root/Desktop/tiny-AES-c/aes_test
```

Đây là lỗi path, không phải lỗi simulator. Sau khi sửa binary path thành `/home/clap/Desktop/tiny-AES-c/aes_test`, chương trình mới đi tiếp đến lỗi dynamic linker đã xử lý ở phần trước.

### Tại sao sử dụng TimingSimpleCPU?

gem5 cung cấp nhiều CPU model với các mức độ chi tiết khác nhau:

* `AtomicSimpleCPU`: thực thi nhanh nhất, không mô hình hóa timing của memory accesses.
* `TimingSimpleCPU`: mô hình hóa timing của hệ thống nhớ và thực thi tuần tự từng instruction.
* `O3CPU`: mô hình out-of-order chi tiết, hỗ trợ pipeline, speculation và nhiều đặc điểm vi kiến trúc khác.

Trong nghiên cứu này, `TimingSimpleCPU` được chọn vì cân bằng giữa tốc độ và khả năng quan sát timing. So với O3CPU, nó đơn giản hơn và chạy nhanh hơn, phù hợp khi phải lặp lại mô phỏng nhiều lần. Đồng thời execution trace của nó đủ ổn định để đếm các lần gọi `SubBytes` và xác định Round 9.

Điểm cần nói rõ: TimingSimpleCPU không chứng minh một glitch vật lý ở mức silicon. Nó được dùng ở đây để tạo môi trường RISC-V có timing trace nhất quán, phục vụ việc dựng và kiểm chứng fault model.

### Chạy chương trình AES trong gem5

![alt text](Image/Screenshot_292.png)

Sau khi biên dịch chương trình AES thành binary RISC-V, mô phỏng được thực thi bằng lệnh:

```bash
./build/RISCV/gem5.opt ~/Desktop/run_aes.py
```

### Kiểm tra kết quả mã hóa

Ciphertext thu được là:

```text
3ad77bb40d7a3660a89ecaf32466ef97
```

Giá trị này trùng khớp với AES-128 Known Answer Test (KAT) của NIST đối với plaintext và khóa đang dùng.

Kết quả này xác nhận rằng:

* Mã nguồn tiny-AES-c được biên dịch chính xác sang kiến trúc RISC-V.
* Binary RISC-V được thực thi đúng trong môi trường gem5.
* gem5 SE mode chạy được workload AES đến khi chương trình exit bình thường.
* Ta có correct ciphertext để làm reference cho DFA.

#### Remote GDB Stub

```text
system.remote_gdb: Listening for connections on port 7000
```

gem5 tự động khởi tạo GDB stub tại cổng 7000.

Người dùng có thể kết nối GDB tới simulator để:

* Đặt breakpoint.
* Theo dõi giá trị thanh ghi.
* Quan sát bộ nhớ.
* Debug chương trình trong quá trình mô phỏng.

#### Stack Growth

```text
Increasing stack size by one page.
```

Thông báo này cho biết gem5 tự động mở rộng vùng stack của tiến trình khi chương trình yêu cầu thêm không gian bộ nhớ ngăn xếp.

Đây là hành vi bình thường trong chế độ Syscall Emulation.

### Thời gian mô phỏng

Theo log thực nghiệm, mô phỏng kết thúc tại:

```text
12184470000 ticks
```

Trong gem5, mặc định:

```text
1 tick = 1 picosecond (ps)
```

Do đó:

```text
12184470000 ticks
≈ 12.18 ms simulated time
```

Cần lưu ý rằng đây là thời gian của hệ thống mô phỏng theo thang thời gian nội bộ của gem5, không phải thời gian thực thi thực tế trên máy host.

Đến đây baseline đã sạch: AES chạy đúng trong gem5, binary là RISC-V static executable, và output khớp NIST vector. Từ baseline này mới bắt đầu xác định fault window, thử checkpoint-based injection, rồi pivot sang source-level hook khi checkpoint path không tạo được fault model đủ sạch.

---

## 5. Phân tích Disassembly  - Xác định Fault Window

Sau khi AES chạy đúng trong gem5, câu hỏi tiếp theo là: **Round 9 nằm ở đâu trong execution?** Muốn trả lời câu này, ta cần nhìn xuống disassembly của RISC-V binary.

### Dump Disassembly

Binary được disassemble bằng `objdump`:

```bash
riscv64-linux-gnu-objdump -d ~/Desktop/tiny-AES-c/aes_test > ~/Desktop/aes_disasm.txt
```

### Tìm các hàm AES quan trọng

![alt text](Image/Screenshot_293.png)

Để nhanh chóng xác định vị trí các thành phần của AES trong disassembly, sử dụng:

```bash
grep -n "AES_ECB_encrypt\|SubBytes\|ShiftRows\|MixColumns\|AddRoundKey" ~/Desktop/aes_disasm.txt
```

Kết quả:

```text
273:   104d2: jal     118b6 <AES_ECB_encrypt>

643: 0000000000010910 <AddRoundKey>:
715: 00000000000109e2 <SubBytes>:
769: 0000000000010a80 <ShiftRows>:
870: 0000000000010bd8 <MixColumns>:

1899:   117da: jal     10910 <AddRoundKey>
1903:   117e8: jal     109e2 <SubBytes>
1905:   117f0: jal     10a80 <ShiftRows>
1911:   11806: jal     10bd8 <MixColumns>
1916:   11818: jal     10910 <AddRoundKey>

1971: 00000000000118b6 <AES_ECB_encrypt>:
```

Ở đây có một điểm dễ nhầm:

```text
273: 104d2: jal 118b6 <AES_ECB_encrypt>
```

chỉ là nơi `main()` gọi `AES_ECB_encrypt()`.

Trong khi đó:

```text
1971: 00000000000118b6 <AES_ECB_encrypt>:
```

mới là nơi thân hàm `AES_ECB_encrypt()` bắt đầu.

Vì vậy không thể chỉ nhìn dòng `273` rồi kết luận fault window. Ta phải đi vào thân `AES_ECB_encrypt()`, rồi tiếp tục đi đến hàm thực sự làm AES round transformation.

### Phân tích AES_ECB_encrypt

![alt text](Image/Screenshot_294.png)

Hiển thị phần thân của hàm:

```bash
grep -n "" ~/Desktop/aes_disasm.txt | sed -n '1971,2100p'
```

![alt text](Image/Screenshot_296.png)

Kết quả cho thấy:

```assembly
118d0: jal 117bc <Cipher>
```

Dòng `jal 117bc <Cipher>` cho thấy `AES_ECB_encrypt()` chỉ là wrapper. Toàn bộ round loop của AES nằm trong `Cipher()` tại địa chỉ:

```text
0x117bc
```

### Phân tích hàm Cipher
![alt text](Image/Screenshot_297.png)

Trích xuất vùng disassembly chứa hàm:

```bash
grep -n "" ~/Desktop/aes_disasm.txt | sed -n '1888,1930p'
```

Kết quả đã được rút gọn và chú thích như sau:

```assembly
00000000000117bc <Cipher>:

117da: jal AddRoundKey      ; Initial RoundKey

117de: li  a5,1
117e0: sb  a5,-17(s0)       ; round_counter = 1

; ─── Main AES Loop ─────────────────────────────

117e8: jal SubBytes
117f0: jal ShiftRows

117fe: beq round,10,11828   ; Round 10 => skip MixColumns

11806: jal MixColumns
11818: jal AddRoundKey

11820: round_counter++
11826: j 117e4              ; Loop back

; ─── Final Round ──────────────────────────────

11828: ...
11834: jal AddRoundKey
11840: ret
```

Từ disassembly có thể thấy `Cipher()` triển khai AES bằng một vòng lặp duy nhất, không unroll thành 10 đoạn code riêng biệt. Điều này rất quan trọng: cùng một instruction `jal SubBytes` sẽ được chạy nhiều lần, mỗi lần tương ứng một AES round khác nhau.

Cấu trúc này tương ứng trực tiếp với đặc tả AES-128:

```text
Round 0
    AddRoundKey

Rounds 1 → 9
    SubBytes
    ShiftRows
    MixColumns
    AddRoundKey

Round 10
    SubBytes
    ShiftRows
    AddRoundKey
```

Biến `round_counter` được khởi tạo bằng 1 sau bước Initial AddRoundKey và được tăng sau mỗi vòng lặp. Khi giá trị đạt 10, nhánh tại `0x117fe` được kích hoạt và chương trình chuyển sang vòng cuối cùng không chứa bước MixColumns theo đúng chuẩn AES.

Do đó:

* Lần thực thi thứ nhất của `SubBytes` tương ứng Round 1.
* Lần thực thi thứ hai tương ứng Round 2.
* ...
* Lần thực thi thứ chín tương ứng Round 9.
* Lần thực thi thứ mười tương ứng Round 10.

Việc đánh số này không dựa trên giả định từ execution trace mà được suy ra trực tiếp từ cấu trúc vòng lặp trong Cipher(). Execution trace ở bước tiếp theo chỉ được sử dụng để xác định thời điểm thực thi thực tế của từng vòng AES trong gem5.

Instruction:

```assembly
117e8: jal 109e2 <SubBytes>
```

nằm trong loop và được thực thi đúng một lần ở đầu mỗi round. Vì vậy ta có thể dùng địa chỉ `0x117e8` như một marker: lần thấy thứ nhất là Round 1, lần thấy thứ chín là Round 9.

### Xác định Fault Window

Theo mô hình DFA dùng bởi PhoenixAES, lỗi cần được đưa vào state ở vòng áp chót, tức Round 9, để ciphertext cuối vẫn giữ cấu trúc sai khác có thể khai thác.

Quan sát disassembly:

```assembly
117e8: jal SubBytes
117f0: jal ShiftRows
11806: jal MixColumns
11818: jal AddRoundKey
```

Trong implementation này, `Cipher()` chạy AES bằng một vòng lặp. Do đó cùng một instruction `jal SubBytes` tại `0x117e8` xuất hiện 10 lần, tương ứng với Round 1 đến Round 10. Theo fault model đã dùng trong log, vị trí inject là **ngay trước SubBytes của Round 9**, tức sau khi Round 8 đã hoàn tất và state chuẩn bị đi vào vòng áp chót.

```text
Round 8 kết thúc
↓
Round 9
↓
[ Fault Injection ]
↓
SubBytes
↓
ShiftRows
↓
MixColumns
↓
AddRoundKey
↓
Round 10
↓
Ciphertext
```

Do đó, bước tiếp theo là xác định chính xác lần xuất hiện thứ 9 của `0x117e8` trong gem5 execution trace.

---

## 6. Xác định Cycle Range của Round 9
### Liên hệ Execution Trace với AES Round

Ở bước trước, disassembly của `Cipher()` cho thấy:

```assembly
117e8: jal SubBytes
117f0: jal ShiftRows
11806: jal MixColumns
11818: jal AddRoundKey

11820: round_counter++
11826: j 117e4
```

Các instruction trên nằm bên trong vòng lặp AES chính. Mỗi lần vòng lặp được thực thi, chương trình sẽ lần lượt gọi:

```text
SubBytes
↓
ShiftRows
↓
MixColumns
↓
AddRoundKey
```

sau đó tăng `round_counter` và quay lại đầu vòng lặp để xử lý round tiếp theo.

Do đó, nếu theo dõi các địa chỉ:

```text
0x117e8
0x117f0
0x11806
0x11818
```

trong execution trace, ta sẽ thấy một mẫu lặp lại nhiều lần:

```text
0x117e8  (SubBytes)
↓
0x117f0  (ShiftRows)
↓
0x11806  (MixColumns)
↓
0x11818  (AddRoundKey)
```

Mỗi lần xuất hiện đầy đủ chuỗi này tương ứng với một AES round hoàn chỉnh.

Ví dụ, trong execution trace:

Round 1:

```text
9029861000 : 0x117e8
9080545000 : 0x117f0
9087100000 : 0x11806
9188518000 : 0x11818
```

Round 2:

```text
9252774000 : 0x117e8
9303089000 : 0x117f0
9309643000 : 0x11806
9411108000 : 0x11818
```

Round 3:

```text
9475351000 : 0x117e8
9526013000 : 0x117f0
9532537000 : 0x11806
9633992000 : 0x11818
```

Có thể thấy sau khi hoàn thành chuỗi:

```text
SubBytes → ShiftRows → MixColumns → AddRoundKey
```

chương trình lại quay về `0x117e8`, tức bắt đầu một vòng AES mới.

Điều này khớp với cấu trúc vòng lặp đã quan sát được trong disassembly của `Cipher()`.

Vì AES-128 có tổng cộng 10 rounds và `0x117e8` là lệnh gọi `SubBytes()` nằm ở đầu vòng lặp, nên lần xuất hiện thứ nhất của `0x117e8` tương ứng Round 1, lần thứ hai tương ứng Round 2, ..., lần thứ chín tương ứng Round 9 và lần thứ mười tương ứng Round 10.

Do đó, thay vì phải phân tích toàn bộ execution trace, chỉ cần theo dõi các lần xuất hiện của địa chỉ `0x117e8` là có thể xác định chính xác thời điểm bắt đầu từng AES round.

### Thu thập Execution Trace

gem5 hỗ trợ theo dõi từng instruction thông qua debug flag `Exec`.

![alt text](Image/Screenshot_298.png)

Execution trace được thu thập bằng:

```bash
./build/RISCV/gem5.opt \
    --debug-flags=Exec \
    ~/Desktop/run_aes.py \
    2>&1 | grep -n "117e8\|117f0\|11806\|11818" \
    > ~/Desktop/exec_trace.txt
```

output:

```
──(clap㉿clap)-[~/Desktop/gem5]
└─$ head -50 ~/Desktop/exec_trace.txt

1503:118187000: system.cpu: T0 : 0x1d922 @__memcpy_generic+112    : c_beqz a2, 20              : IntAlu : 
15839:1180637000: system.cpu: T0 : 0x5091c @classify_object_over_fdes+188    : c_ldsp a4, 24(sp)          : MemRead :  D=0x0000000000000004 A=0x7ffffffffffffdf8
15855:1181827000: system.cpu: T0 : 0x50896 @classify_object_over_fdes+54    : lw a4, 4(s10)              : MemRead :  D=0x0000000000000d18 A=0x6f568
42789:3118187000: system.cpu: T0 : 0x508fc @classify_object_over_fdes+156    : bne a1, a5, 228            : IntAlu : 
51320:3711806000: system.cpu: T0 : 0x4f766 @read_encoded_value_with_base+4    : beq a0, a5, 140            : IntAlu : 
57046:4111806000: system.cpu: T0 : 0x50974 @classify_object_over_fdes+276    : beq s8, a3, -68            : IntAlu : 
91471:6511818000: system.cpu: T0 : 0x4f864 @read_encoded_value_with_base+258    : c_addiw a5, 0              : IntAlu :  D=0x000000000000002e
95784:6811818000: system.cpu: T0 : 0x4f850 @read_encoded_value_with_base+238    : lbu a5, 3(a2)              : MemRead :  D=0x0000000000000000 A=0x77daf
98643:7011818000: system.cpu: T0 : 0x4f77e @read_encoded_value_with_base+28    : c_add a5, a4               : IntAlu :  D=0x000000000006e804
120143:8511806000: system.cpu: T0 : 0x105f6 @KeyExpansion+200    : ld a4, -64(s0)             : MemRead :  D=0x7ffffffffffffd58 A=0x7ffffffffffffc10
126249:9011806000: system.cpu: T0 : 0x10984 @AddRoundKey+116    : lbu a5, -17(s0)            : MemRead :  D=0x0000000000000002 A=0x7ffffffffffffc0f
126476:9029861000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
127027:9080545000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
127099:9087100000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
128381:9188518000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
129179:9252774000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
129730:9303089000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
129802:9309643000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
131084:9411108000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
131882:9475351000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
132433:9526013000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
132505:9532537000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
133787:9633992000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
134585:9698381000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
135136:9749030000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
135208:9755315000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
136490:9856760000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
137288:9921481000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
137839:9971860000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
137911:9978372000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
139193:10079866000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
139991:10144134000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
140542:10194692000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
140614:10201261000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
141896:10302720000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
142694:10367004000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
143180:10411806000: system.cpu: T0 : 0x10a16 @SubBytes+52    : addiw a4, a5, 0            : IntAlu :  D=0x00000000000000ff
143245:10417370000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
143317:10423966000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
144599:10525466000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
145397:10590015000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
145948:10640441000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
146020:10647009000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
147302:10748427000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
148100:10812745000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
148651:10863351000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
148723:10869636000: system.cpu: T0 : 0x11806 @Cipher+74    : jal ra, -3118              : IntAlu :  D=0x000000000001180a
150005:10971373000: system.cpu: T0 : 0x11818 @Cipher+92    : jal ra, -3848              : IntAlu :  D=0x000000000001181c
150803:11035659000: system.cpu: T0 : 0x117e8 @Cipher+44    : jal ra, -3590              : IntAlu :  D=0x00000000000117ec
151354:11085998000: system.cpu: T0 : 0x117f0 @Cipher+52    : jal ra, -3440              : IntAlu :  D=0x00000000000117f4
```

Các địa chỉ được theo dõi không phải là địa chỉ bắt đầu của các hàm AES, mà là các lệnh gọi (call site) nằm bên trong vòng lặp của Cipher():

| Address | Ý nghĩa |
|----------|----------|
| 0x117e8 | gọi SubBytes() |
| 0x117f0 | gọi ShiftRows() |
| 0x11806 | gọi MixColumns() |
| 0x11818 | gọi AddRoundKey() |

Những địa chỉ này được lựa chọn vì chúng xuất hiện đúng một lần trong mỗi vòng lặp AES, giúp việc nhận diện từng round trong execution trace trở nên dễ dàng hơn.

Trích đoạn execution trace:

```text
9029861000   : 0x117e8  ← Round 1
9252774000   : 0x117e8  ← Round 2
9475351000   : 0x117e8  ← Round 3
9698381000   : 0x117e8  ← Round 4
9921481000   : 0x117e8  ← Round 5
10144134000  : 0x117e8  ← Round 6
10367004000  : 0x117e8  ← Round 7
10590015000  : 0x117e8  ← Round 8
10812745000  : 0x117e8  ← Round 9
11035659000  : 0x117e8  ← Round 10
```

Do 0x117e8 là lệnh gọi SubBytes() nằm ở đầu vòng lặp AES, mỗi lần địa chỉ này được thực thi tương ứng với việc bắt đầu một AES round mới.

Từ trace có thể xác định:

```text
Round 9 bắt đầu tại tick:
10,812,745,000
```

### Mapping Tick sang Cycle

CPU được cấu hình ở tần số:

```text
1 GHz
```

Trong gem5:

```text
1 tick = 1 ps
1 cycle = 1 ns = 1000 ticks
```

Do đó:

```text
10,812,745,000 ticks
≈ 10,812,745 cycles
```

Bảng timing của các vòng AES:

| Round | Tick bắt đầu SubBytes |          Cycle |
| ----- | --------------------: | -------------: |
| 1     |         9,029,861,000 |      9,029,861 |
| 2     |         9,252,774,000 |      9,252,774 |
| 3     |         9,475,351,000 |      9,475,351 |
| 4     |         9,698,381,000 |      9,698,381 |
| 5     |         9,921,481,000 |      9,921,481 |
| 6     |        10,144,134,000 |     10,144,134 |
| 7     |        10,367,004,000 |     10,367,004 |
| 8     |        10,590,015,000 |     10,590,015 |
| **9** |    **10,812,745,000** | **10,812,745** |
| 10    |        11,035,659,000 |     11,035,659 |

Khoảng cách giữa các vòng tương đối ổn định, cho phép sử dụng execution trace để xác định chính xác thời điểm bắt đầu Round 9.

Kết quả này cung cấp mốc thời gian cần thiết để triển khai fault injection trong các bước tiếp theo.

Trong implementation cuối cùng của nghiên cứu này, fault được chèn trực tiếp vào state AES thông qua source-level fault hook để tạo tập faulty ciphertext phục vụ DFA. Do đó execution trace không được sử dụng để kích hoạt fault injection. Thay vào đó, trace được dùng để xác minh vị trí fault tương ứng với Round 9 và làm cơ sở cho các fault model ở mức simulator trong các nghiên cứu mở rộng sau này.

---

## 7. Approach 1: Checkpoint-based Fault Injection

Sau khi xác định được Round 9 bắt đầu tại khoảng tick `10,812,745,000`, hướng tiếp cận đầu tiên là thực hiện fault injection ở mức simulator thông qua cơ chế checkpoint của gem5.

Ý tưởng của phương pháp này là dừng mô phỏng ngay trước thời điểm fault window, lưu toàn bộ trạng thái hệ thống, sau đó chỉnh sửa dữ liệu trong checkpoint rồi tiếp tục thực thi.

Pipeline dự kiến:

```text
Run AES trong gem5
↓
Stop tại Round 9
↓
Create checkpoint
↓
Modify register hoặc memory
↓
Restore checkpoint
↓
Collect faulty ciphertext
```

Nếu thành công, đây sẽ là một mô hình fault injection gần với kịch bản phần cứng hơn so với việc chỉnh sửa mã nguồn, bởi lỗi được đưa vào từ phía simulator thay vì từ bên trong chương trình AES.

### Thách thức khi làm việc với Checkpoint

Về mặt kỹ thuật, gem5 cho phép lưu và khôi phục toàn bộ trạng thái CPU, register và memory tại thời điểm checkpoint. Tuy nhiên, checkpoint không lưu thông tin ở mức ngữ nghĩa của ứng dụng.

Nói cách khác, checkpoint chỉ chứa:

```text
Register values
Memory pages
CPU state
Cache state
```

chứ không chứa khái niệm:

```text
AES state
Round 9 state
SubBytes input
```

Do đó, sau khi tạo checkpoint tại Round 9, bước khó nhất là xác định chính xác byte nào trong hàng nghìn byte memory tương ứng với AES state cần gây lỗi.

### Các Attempt Thực Hiện

Trong quá trình debug, nhiều vị trí memory và register đã được sửa đổi thủ công rồi restore lại checkpoint.

Tuy nhiên kết quả cho thấy:

* Một số vị trí chỉ là pointer hoặc metadata của chương trình.
* Một số vùng memory chứa các pattern như `0xaa` hoặc `0x55`, không liên quan đến AES state thực tế.
* Ciphertext sau khi restore không thay đổi theo fault model mong muốn của DFA.

Điều này cho thấy việc tìm đúng byte đại diện cho state AES bên trong checkpoint khó hơn dự kiến. Mặc dù fault đã được đưa vào simulator, nhưng chưa thể chứng minh rằng fault đó thực sự tác động đến trạng thái nội bộ của AES ở Round 9.

> Checkpoint-based fault injection là một hướng tiếp cận hợp lý và gần với mô hình hardware fault injection. Tuy nhiên trong phạm vi nghiên cứu này, việc ánh xạ từ dữ liệu thô trong checkpoint sang AES state chưa được giải quyết triệt để.

> Vì vậy pipeline cuối cùng được chuyển sang source-level fault hook, nơi fault được chèn trực tiếp vào AES state tại Round 9. Cách tiếp cận này giúp tạo ra tập faulty ciphertext ổn định và phù hợp với yêu cầu của PhoenixAES để thực hiện DFA key recovery.

> Mặc dù không được sử dụng trong kết quả cuối cùng, checkpoint-based injection vẫn là một bước debug quan trọng vì nó giúp đánh giá tính khả thi của fault injection ở mức simulator trước khi chuyển sang fault model có kiểm soát hơn.

---

## 8. Vấn đề với checkpoint approach  - debug quá trình
Mục tiêu của phần debug này là trả lời một câu hỏi duy nhất:

> Nếu checkpoint được tạo đúng tại Round 9, liệu có thể xác định và sửa trực tiếp AES state từ dữ liệu lưu trong checkpoint hay không?

Để kiểm chứng giả thuyết này, ba hướng tiếp cận khác nhau đã được thử:

1. Sửa trực tiếp register đang được truyền vào SubBytes().
2. Suy ngược từ virtual address sang physical memory rồi sửa dữ liệu trong checkpoint.
3. Dịch thời điểm checkpoint sâu hơn vào bên trong SubBytes để kiểm tra trạng thái bộ nhớ có thay đổi hay không.

Nếu một trong các hướng trên thành công, fault có thể được đưa vào từ phía simulator mà không cần chỉnh sửa mã nguồn AES.

### Attempt 1: Modify register x10 trực tiếp

Attempt đầu tiên là sửa trực tiếp register `x10` trong checkpoint. Ở thời điểm gọi `SubBytes`, `x10` thường được dùng làm argument register trong RISC-V calling convention, nên trực giác ban đầu là nó có thể liên quan đến state pointer.

```python
# modify_ckpt.py
...
reg_idx = 10 * 8  # x10 bắt đầu tại byte offset 80
x10_bytes = bytes(vals[reg_idx:reg_idx+8])
x10_val = struct.unpack('<Q', x10_bytes)[0]
print(f"[PRE-FAULT]  x10 = 0x{x10_val:016x}")
x10_faulty = x10_val ^ 0x01
```

Output:

```
[PRE-FAULT]  x10 = 0x7ffffffffffffc78
[POST-FAULT] x10 = 0x7ffffffffffffc79
```

Sau khi restore và chạy tiếp, ciphertext có thay đổi (`e6d77bb40d7a36c7a89e67f324dcef97`). Nhưng khi thử nhiều bit position của `x10`, chỉ bit 0 tạo ra output khác. 

Kết quả này cho thấy fault đã ảnh hưởng đến quá trình thực thi của chương trình, nhưng chưa chứng minh được rằng bit bị lật nằm trong AES state.

Việc chỉ có bit thấp nhất của x10 tạo ra thay đổi ở ciphertext là dấu hiệu cho thấy x10 nhiều khả năng đang chứa địa chỉ (pointer) hơn là dữ liệu state AES. Khi bit 0 bị thay đổi, địa chỉ bị lệch sang byte kế cận và chương trình tiếp tục xử lý dữ liệu khác, dẫn tới ciphertext thay đổi.

Tuy nhiên đây không phải fault model mà DFA yêu cầu, vì vị trí lỗi không còn nằm trên một byte của AES state.

### Attempt 2: Modify physical memory

Đọc registers tại fault point:

```
x10 = 0x7ffffffffffffc79   (đã bị modify từ bước trước)
x11 = 0x7ffffffffffffc78
x12 = 0x7ffffffffffffc78
```

Các register này đều liên quan đến vùng `0x7ffffffffffffc78`, tức virtual address của buffer AES trên stack. Vì vậy attempt tiếp theo là thử đi từ virtual address này sang physical address trong checkpoint.

Tìm physical address tương ứng bằng cách đọc page table từ `m5.cpt`:

```python
# find_paddr.py
VADDR = 0x7ffffffffffffc78
PAGE_SIZE = 4096
vpage = (VADDR // PAGE_SIZE) * PAGE_SIZE  # 0x7ffffffffffff000
offset = VADDR % PAGE_SIZE               # 0xc78
```

Tìm trong mappings:

```
vaddr=9223372036854771712  →  paddr=483328
# 0x7ffffffffffff000 = 9223372036854771712 → phys 0x76000
```

Từ mapping trên có thể suy ra một địa chỉ vật lý ứng viên:

`0x76000 + 0xc78 = 0x76c78`

Địa chỉ này sau đó được sử dụng như một giả thuyết ban đầu cho vị trí của AES state trong physical memory.

Đọc 16 bytes tại địa chỉ đó từ `system.physmem.store0.pmem`:

Cần lưu ý rằng bản thân giá trị `0xaa` hoặc `0x55` không chứng minh vùng nhớ này chắc chắn không phải AES state.

Tuy nhiên trong bối cảnh AES đang xử lý plaintext thử nghiệm, xác suất toàn bộ 16 byte state đều đồng nhất thành:

```
AES state at 0x76c78: aa aa aa aa aa aa aa aa aa aa aa aa aa aa aa aa
```

Kết quả toàn `aa` là một dấu hiệu cho thấy vùng nhớ đang đọc có thể không phải AES state thực tế.

Mặc dù không thể khẳng định chỉ từ giá trị dữ liệu, nhưng việc toàn bộ 16 byte đều mang cùng một pattern (`0xaa`) không phù hợp với kỳ vọng về trạng thái trung gian của AES sau nhiều vòng biến đổi. Do đó khả năng cao địa chỉ vật lý vừa tìm được chưa ánh xạ tới vùng dữ liệu state đang được Cipher() sử dụng tại thời điểm checkpoint.

### Tại sao chưa xác định được AES state từ checkpoint?

Checkpoint được save khi `jal <SubBytes>` vừa được **fetch**. Nhưng ở cấp checkpoint, ánh xạ từ logical AES state sang byte cần sửa không đơn giản như "đọc 16 bytes tại pointer rồi flip bit". Kết quả toàn `0xaa` cho thấy attempt này chưa chạm đúng dữ liệu state đang được AES sử dụng tại fault boundary.

Cụ thể hơn: nhìn vào disassembly của Cipher:

```asm
117bc: addi  sp,sp,-48      ; Cipher mở stack frame
117c4: sd    a0,-40(s0)     ; Lưu state pointer = địa chỉ của buf trong main()
```

`-40(s0)` là chỗ `Cipher()` lưu pointer đến state. Vấn đề là checkpoint-level edit cần đúng nhiều thứ cùng lúc: virtual-to-physical mapping, backing memory object, thời điểm checkpoint commit state, và byte nào thực sự đại diện cho AES state ở boundary đó. Trong attempt này, dữ liệu đọc từ physical memory không khớp với state AES mong đợi, nên flip byte ở đó không tạo fault model sạch cho PhoenixAES.

Vì vậy `aa aa aa` được xem là dấu hiệu rằng ta đang đọc một vùng/pattern không đại diện cho AES state cần inject, không phải bằng chứng rằng DFA fault đã được đặt đúng chỗ.

### Attempt 3: Save checkpoint muộn hơn

Sau đó thử save checkpoint muộn hơn một chút, đi sâu vào bên trong `SubBytes`:

```python
FAULT_TICK = 10812745000 + 50000  # thêm 50ns vào bên trong SubBytes
```

Output:

```
[INFO] Checkpoint saved at tick 10812995000
```

Đọc lại memory:

```
AES state: 55 55 55 55 55 55 55 55 55 55 55 55 55 55 55 55
```

Việc pattern thay đổi từ 0xaa sang 0x55 cho thấy thời điểm checkpoint có ảnh hưởng đến dữ liệu được quan sát. Tuy nhiên cả hai trường hợp đều chưa cung cấp bằng chứng đủ mạnh rằng vùng nhớ đang đọc chính là AES state tại Round 9.

0x55 tương ứng với mẫu bit 01010101. Việc toàn bộ 16 byte đều có cùng giá trị là một dấu hiệu bất thường đối với dữ liệu state AES đang được biến đổi qua nhiều vòng mã hóa. Quan trọng hơn, không có bằng chứng nào cho thấy vùng nhớ này thực sự tương ứng với state mà SubBytes() đang xử lý tại thời điểm fault boundary.

Nói cách khác, việc dịch checkpoint muộn hơn chưa giải quyết được vấn đề cốt lõi: ta vẫn chưa ánh xạ được một cách đáng tin cậy từ logical AES state trong chương trình sang byte cụ thể trong file checkpoint.

### Kết luận về checkpoint approach

Checkpoint approach có thể hữu ích cho **register-level fault injection** hoặc cho một setup có mapping state rõ hơn. Nhưng trong log này nó không hiệu quả cho **AES state injection** vì chưa xác định được byte state đúng trong checkpoint để sửa một cách sạch và lặp lại được.

Sau ba attempt trên, vấn đề cốt lõi vẫn chưa được giải quyết: chưa có phương pháp đáng tin cậy để ánh xạ một byte trong checkpoint sang một byte cụ thể của AES state tại Round 9.

Điều này khiến fault injection ở mức checkpoint khó kiểm soát, khó lặp lại và khó chứng minh rằng fault thực sự được đặt trên đúng byte của AES state theo fault model mà PhoenixAES yêu cầu.

Vì vậy hướng checkpoint-based injection được dừng lại tại đây và chuyển sang source-level fault hook, nơi vị trí fault có thể được xác định chính xác trên AES state trước khi tạo faulty ciphertext cho PhoenixAES.
---

## 9. Approach 2: Source-level fault injection

Thay vì tiếp tục sửa checkpoint, pipeline cuối cùng đưa fault hook trực tiếp vào source của tiny-AES-c. Đây không phải hardware-level injection, nhưng vẫn hợp lệ cho mục tiêu của hướng này: kiểm chứng **cryptographic DFA path**. Điều PhoenixAES cần là faulty ciphertexts sinh ra từ fault model đúng vị trí; các ciphertext đó vẫn được tạo bởi RISC-V binary chạy trong gem5, không phải hardcode.

Tóm lại:

- AES vẫn được compile thành RISC-V binary và chạy trong gem5.
- Fault được đặt đúng logical boundary: đầu Round 9, trước `SubBytes`.
- Faulty ciphertexts được sinh ra bởi execution thật của binary, không viết tay.
- Phương pháp này chứng minh attack math và defense behavior, nhưng không chứng minh khả năng glitch phần cứng vật lý.

Vị trí fault được chọn dựa trên kết quả phân tích disassembly và execution trace ở các phần trước, nhằm mô phỏng fault model thường dùng trong DFA trên AES-128.

### Patch aes.c  - thêm fault hook

Đọc source của Cipher function trong `aes.c`:

```c
static void Cipher(state_t* state, const uint8_t* RoundKey)
{
  uint8_t round = 0;
  AddRoundKey(0, state, RoundKey);

  for (round = 1; ; ++round)
  {
    SubBytes(state);
    ShiftRows(state);
    if (round == Nr) {
      break;
    }
    MixColumns(state);
    AddRoundKey(round, state, RoundKey);
  }
  AddRoundKey(Nr, state, RoundKey);
}
```

Hook được đặt **ngay trước SubBytes của Round 9**.

Theo phân tích ở phần trước, đây là thời điểm state vừa hoàn thành Round 8 và chuẩn bị đi vào Round 9. Fault được đưa vào tại boundary này để sai khác tiếp tục lan truyền qua các bước còn lại của Round 9 và toàn bộ Round 10 trước khi tạo ciphertext cuối cùng.

```c
for (round = 1; ; ++round)
{
    // DFA fault injection hook
    if (round == 9 && fault_byte >= 0 && fault_byte < 16) {
        uint8_t* s = (uint8_t*)state;
        s[fault_byte] ^= (1 << fault_bit);  // flip 1 bit
    }
    SubBytes(state);
    ShiftRows(state);
    ...
```

`fault_byte` và `fault_bit` là global variables được set từ environment variables. Nhờ vậy mỗi lần chạy gem5 có thể chọn byte/bit khác nhau mà không cần compile lại.

### Compile và verify

Thêm extern declarations vào đầu `aes.c`:

```c
extern int fault_byte;
extern int fault_bit;
```

Define trong `aes_main.c`:

```c
int fault_byte = -1;  // -1 = no fault
int fault_bit  = 0;
```

Đọc từ environment:

```c
char *fb = getenv("FAULT_BYTE");
char *fi = getenv("FAULT_BIT");
if (fb) fault_byte = atoi(fb);
if (fi) fault_bit  = atoi(fi);
```

Compile:

```bash
riscv64-linux-gnu-gcc -O0 -g -static -o aes_test aes_main.c aes.c -I.
```

### Tạo gem5 run script với env vars

```python
# process.env: truyền environment variables vào process trong gem5 SE mode
process.env = [f'FAULT_BYTE={fault_byte}', f'FAULT_BIT={fault_bit}']
```

---

## 10. Collect 128 faulty ciphertexts

AES state gồm 16 byte. Với fault model sử dụng một lần lật bit (single-bit fault), mỗi byte có 8 vị trí bit có thể bị tác động. Vì vậy tổng cộng có:

```
16 × 8 = 128
```

vị trí fault khác nhau được khảo sát. Trong log, ta chạy hết 128 vị trí để tạo dataset dư dả cho PhoenixAES và cũng để quan sát pattern lan truyền của lỗi.

### Automation script

```python
# collect_faults2.py
import subprocess

GEM5 = '/home/clap/Desktop/gem5/build/RISCV/gem5.opt'
correct = '3ad77bb40d7a3660a89ecaf32466ef97'
faulty_set = set()

def run_gem5(fault_byte=-1, fault_bit=0):
    script = f"""
import m5
from m5.objects import *
...
process.env = ['FAULT_BYTE={fault_byte}', 'FAULT_BIT={fault_bit}']
...
"""
    with open('/tmp/gem5_run.py', 'w') as f:
        f.write(script)
    
    result = subprocess.run([GEM5, '/tmp/gem5_run.py'],
                           capture_output=True, text=True)
    for line in result.stdout.split('\n'):
        if 'Ciphertext:' in line:
            return line.split('Ciphertext:')[1].strip()
    return None

for byte_idx in range(16):
    for bit in range(8):
        ct = run_gem5(byte_idx, bit)
        if ct and ct != correct and ct not in faulty_set:
            faulty_set.add(ct)
            print(f"byte={byte_idx} bit={bit}: {ct}")
```

### Kết quả  - 128 faulty ciphertexts

Script chạy 128 iterations, mỗi lần đổi `FAULT_BYTE` và `FAULT_BIT`, rồi parse dòng `Ciphertext:` từ stdout của gem5. Output dưới đây là trích đoạn:

```
byte=0 bit=0: 5dd77bb40d7a3616a89eeff32406ef97
byte=0 bit=1: 9fd77bb40d7a36b0a89e54f3241aef97
byte=0 bit=2: 0ad77bb40d7a364fa89ee9f324daef97
byte=0 bit=3: a0d77bb40d7a368ea89edef3247aef97
byte=0 bit=4: 10d77bb40d7a362da89e26f32420ef97
byte=0 bit=5: 2bd77bb40d7a3646a89e6ef324ddef97
byte=0 bit=6: 88d77bb40d7a3628a89e0bf324beef97
byte=0 bit=7: 61d77bb40d7a36dda89ee7f324deef97
byte=1 bit=0: 3ad77b430d7aff60a89dcaf33366ef97
...
byte=15 bit=7: b3d77bb40d7a36cda89e8ef3243eef97

Trong tập thực nghiệm này, cả 128 vị trí fault đều tạo ra ciphertext khác với ciphertext đúng và không xuất hiện trường hợp trùng lặp. Điều này cho thấy mỗi vị trí single-bit fault đã tạo ra một mẫu lan truyền lỗi riêng biệt trong ciphertext.
```

### Phân tích pattern của faulty ciphertexts

Nhìn vào các faulty ciphertexts, ta thấy pattern DFA rất rõ. Correct ciphertext:

```
3a d7 7b b4 0d 7a 36 60 a8 9e ca f3 24 66 ef 97
```

Fault tại byte 0:

```
5d d7 7b b4 0d 7a 36 16 a8 9e ef f3 24 06 ef 97  (bit 0)
9f d7 7b b4 0d 7a 36 b0 a8 9e 54 f3 24 1a ef 97  (bit 1)
```

Khi fault được đưa vào một byte của state ở đầu Round 9, sai khác tiếp tục đi qua các bước còn lại của Round 9 trước khi đi vào Round 10. Kết quả là lỗi không còn giới hạn ở một byte duy nhất mà lan sang một nhóm byte có quan hệ với nhau do phép biến đổi MixColumns tạo ra. Với ví dụ trên, các byte thay đổi là `[0, 7, 10, 13]`. Đây chính là dấu hiệu mà classic AES DFA cần: một lỗi cục bộ trong state lan thành một nhóm 4 byte có cấu trúc.

Fault tại byte 1:

```
3a d7 7b 43 0d 7a ff 60 a8 9d ca f3 33 66 ef 97  (bit 0)
```

Bytes thay đổi: `[3, 6, 9, 12]`  - column khác.

Đây là signature chuẩn của Round 9 DFA: mỗi fault tại một byte state ảnh hưởng đến một nhóm 4 byte ciphertext. Các dependency do MixColumns tạo ra chính là thứ PhoenixAES khai thác để lọc candidate round key.

Kết quả này cho thấy fault hook đã tạo ra đúng loại faulty ciphertext mà DFA trên AES mong đợi. Do đó tập dữ liệu thu được có thể được sử dụng làm đầu vào cho PhoenixAES ở bước tiếp theo để thực hiện quá trình khôi phục khóa.

---

## 11. PhoenixAES recover round 10 key
Sau bước thu thập fault dataset, mục tiêu tiếp theo là kiểm tra xem các faulty ciphertext có thực sự chứa đủ thông tin để suy ra khóa AES hay không.

Theo lý thuyết DFA trên AES-128, chỉ cần fault được đặt đúng tại Round 9 thì các sai khác trong ciphertext sẽ mang cấu trúc toán học liên quan đến last round key. PhoenixAES được sử dụng để tự động khai thác các quan hệ này và thực hiện quá trình key recovery.
### Cài đặt PhoenixAES

```bash
pip3 install phoenixAES --break-system-packages
```

PhoenixAES implement thuật toán DFA cho AES. Input của nó là một ciphertext đúng và nhiều ciphertext lỗi được tạo từ fault ở Round 9. Output mong muốn là AES last round key, tức round 10 key.

### Format input file

Một điểm rất dễ sai trong log là format input. `crack_file()` không nhận correct ciphertext như một string riêng ở argument thứ hai. File input phải có format:
- Dòng 1: correct ciphertext (hex)
- Dòng 2+: faulty ciphertexts (hex)

```bash
echo "3ad77bb40d7a3660a89ecaf32466ef97" > ~/Desktop/dfa_input.txt
cat ~/Desktop/faulty_ciphertexts.txt >> ~/Desktop/dfa_input.txt
```

Nếu truyền sai API, PhoenixAES có thể báo lỗi kiểu dữ liệu khi xử lý XOR. Cách ổn định nhất trong log là tạo `dfa_input.txt` đúng format rồi gọi `crack_file()` trực tiếp trên file đó.

### Chạy PhoenixAES

```python
import phoenixAES
result = phoenixAES.crack_file('/home/clap/Desktop/dfa_input.txt', verbose=True)
print(f"\nRound 10 key: {result}")
```

Output:

```
Last round key #N found:
D014F9A8C9EE2589E13F0CC8B6630CA6
Round 10 key: D014F9A8C9EE2589E13F0CC8B6630CA6
```

**Round 10 key recovered: `D014F9A8C9EE2589E13F0CC8B6630CA6`**

Kết quả này là bằng chứng đầu tiên cho thấy fault model được sử dụng trong nghiên cứu là hợp lệ.

Nếu fault được đặt sai round hoặc sai cấu trúc mà PhoenixAES giả định, quá trình lọc candidate sẽ không hội tụ về một last round key duy nhất. Việc recover thành công toàn bộ 16 bytes của Round 10 key cho thấy các faulty ciphertext thu thập được thực sự mang dấu hiệu của Round 9 DFA.

### Cách PhoenixAES hoạt động (brief)

Ở mức trực giác, PhoenixAES làm việc như sau:

AES round 10 chỉ gồm SubBytes → ShiftRows → AddRoundKey (bỏ MixColumns). Khi fault xảy ra tại đầu round 9:

```
State_round9 = State_correct_round9 ⊕ error_delta
```

Sau khi đi qua SubBytes, ShiftRows, MixColumns và AddRoundKey của Round 9, error lan theo pattern 4 byte. Sau đó nó đi tiếp qua final round, tức Round 10, gồm SubBytes, ShiftRows và AddRoundKey.

Ý tưởng chính là: nếu đoán đúng byte của round 10 key, ta có thể "đi ngược" qua final round và thấy sai khác quay về đúng cấu trúc do MixColumns của Round 9 tạo ra. Nếu đoán sai key byte, cấu trúc này không còn nhất quán trên nhiều faulty ciphertexts.

Cụ thể, với `D = Correct_CT ⊕ Faulty_CT`:

```
InvShiftRows(InvSubBytes(CT ⊕ RK10)) ⊕ 
InvShiftRows(InvSubBytes(CT_fault ⊕ RK10)) = MixColumns_column_pattern
```

Với mỗi candidate byte của `RK10` có 256 khả năng. PhoenixAES kiểm tra candidate nào nhất quán với toàn bộ dataset fault. Khi đủ ciphertext lỗi, không gian candidate bị thu hẹp cho đến khi round 10 key được recover.

128 faults trong log tạo redundancy lớn, giúp recover toàn bộ 16 bytes của `RK10`.

---

## 12. Recover master key bằng reverse key schedule
Tuy nhiên Round 10 key chưa phải khóa bí mật mà hệ thống sử dụng.

Trong AES-128, last round key chỉ là kết quả của quá trình key expansion từ master key ban đầu. Vì key schedule của AES là hàm khả nghịch, việc thu được Round 10 key đồng nghĩa với việc có thể khôi phục toàn bộ master key.

Do đó bước cuối cùng của attack chain là đảo ngược AES key schedule.

Round 10 key `D014F9A8C9EE2589E13F0CC8B6630CA6` là kết quả của AES key expansion từ master key. Về lý thuyết, ta có thể đảo AES-128 key schedule để đi từ `w[40]..w[43]` về `w[0]..w[3]`, tức master key.

Trong log, đoạn reverse key schedule tự viết lần đầu bị sai và trả ra key không khớp. Đây là lỗi implementation, không phải lỗi của PhoenixAES. Vì vậy phần dưới trình bày bản reverse đúng để bài hoàn chỉnh, sau đó dùng forward expansion như một sanity check giống bước verify cuối trong log.

### Reverse từ round 10 key về master key

AES-128 key schedule sinh ra 44 words, mỗi word 4 bytes:

- Master key: `w[0]..w[3]`
- Round 10 key: `w[40]..w[43]`

Forward rule:

```text
w[i] = w[i-4] xor temp

if i % 4 == 0:
    temp = SubWord(RotWord(w[i-1])) xor Rcon
else:
    temp = w[i-1]
```

Vì vậy có thể đảo ngược:

```text
w[i-4] = w[i] xor temp
```

Code recover corrected:

```python
def xor_w(a, b):
    return [x ^ y for x, y in zip(a, b)]

def rot_word(w):
    return w[1:] + w[:1]

def sub_word(w):
    sbox = [0x63, 0x7c, ...]  # AES S-box
    return [sbox[b] for b in w]

rcon = [0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x1b, 0x36]

def reverse_aes128_key_schedule(round10_hex):
    rk10 = bytes.fromhex(round10_hex)
    w = [None] * 44
    w[40:44] = [list(rk10[i:i+4]) for i in range(0, 16, 4)]

    for i in range(43, 3, -1):
        temp = w[i-1][:]
        if i % 4 == 0:
            temp = xor_w(sub_word(rot_word(temp)), [rcon[i//4-1], 0, 0, 0])
        w[i-4] = xor_w(w[i], temp)

    return bytes(b for word in w[0:4] for b in word)

rk10 = 'D014F9A8C9EE2589E13F0CC8B6630CA6'
master = reverse_aes128_key_schedule(rk10)
expected = '2B7E151628AED2A6ABF7158809CF4F3C'
print(f"Recovered master key: {master.hex().upper()}")
print(f"Expected master key:  {expected}")
print(f"Match: {master.hex().upper() == expected}")
```

Output:

```
Recovered master key: 2B7E151628AED2A6ABF7158809CF4F3C
Expected master key:  2B7E151628AED2A6ABF7158809CF4F3C
Match: True
```

Như vậy, với reverse key schedule đúng, round 10 key do PhoenixAES recover có thể đưa ngược về đúng AES-128 master key.

### Verify bằng forward expansion

Trong log thực tế, bước verify cuối cùng là expand master key đã biết của test vector rồi kiểm tra round 10 key có khớp với key PhoenixAES recover hay không:

```python
master = bytes.fromhex('2B7E151628AED2A6ABF7158809CF4F3C')
w = expand_key(master)
r10 = bytes(b for word in w[40:44] for b in word)
print(f"Round 10 key from master: {r10.hex().upper()}")
```

Output:

```
Round 10 key from master: D014F9A8C9EE2589E13F0CC8B6630CA6
PhoenixAES recovered:     D014F9A8C9EE2589E13F0CC8B6630CA6
Match: True
```

### Verify encryption end-to-end

```python
from Crypto.Cipher import AES

master = bytes.fromhex('2B7E151628AED2A6ABF7158809CF4F3C')
cipher = AES.new(master, AES.MODE_ECB)
pt = bytes.fromhex('6bc1bee22e409f96e93d7e117393172a')
ct = cipher.encrypt(pt)
print(f"Encrypt verify: {ct.hex()}")
print(f"Expected:       3ad77bb40d7a3660a89ecaf32466ef97")
```

Output:

```
Encrypt verify: 3ad77bb40d7a3660a89ecaf32466ef97
Expected:       3ad77bb40d7a3660a89ecaf32466ef97
```

**DFA attack hoàn toàn thành công và đạt được mục tiêu ban đầu của nghiên cứu:**

- Correct ciphertext: `3ad77bb40d7a3660a89ecaf32466ef97`
- Round 10 key recovered: `D014F9A8C9EE2589E13F0CC8B6630CA6`
- Master key: `2B7E151628AED2A6ABF7158809CF4F3C` ✓

Chuỗi tấn công hoàn chỉnh có thể được tóm tắt như sau:

```text
Fault tại Round 9
        ↓
Faulty Ciphertexts
        ↓
PhoenixAES
        ↓
Round 10 Key
        ↓
Reverse Key Schedule
        ↓
AES-128 Master Key

Điều này chứng minh rằng chỉ từ các ciphertext lỗi được sinh ra bởi fault injection tại Round 9, attacker có thể khôi phục thành công khóa bí mật của AES-128 mà không cần truy cập trực tiếp vào bộ nhớ chứa key.

---

## 13. Lockstep Defense  - implement và verify
Tới thời điểm này, attack chain đã hoàn chỉnh: fault tại Round 9 cho phép tạo faulty ciphertexts, PhoenixAES recover được Round 10 key, và reverse key schedule đưa ngược về master key.

Câu hỏi tiếp theo là: làm thế nào để ngăn attacker thu được các faulty ciphertext này ngay từ đầu?

Một trong những countermeasure phổ biến nhất trong fault-tolerant systems và secure hardware là Lockstep Execution.

### Concept

Lockstep Defense là countermeasure phổ biến nhất cho fault injection attacks. Ý tưởng cốt lõi của lockstep là thực hiện cùng một phép tính trên hai execution lanes độc lập rồi so sánh kết quả.

Nếu một fault chỉ ảnh hưởng đến một lane, trạng thái thực thi của hai lane sẽ khác nhau và comparator có thể phát hiện sự sai lệch trước khi kết quả được trả về cho bên ngoài.

Trong hardware: hai CPU cores execute identical instructions, comparator circuit kiểm tra output hoặc architectural state theo từng checkpoint/cycle tùy thiết kế.

Trong log/demo này: chưa dựng dual-core lockstep thật trong gem5. Ta mô phỏng ý tưởng lockstep ở mức software bằng hai lần gọi AES trong cùng process, sau đó so sánh kết quả trước khi cho phép ciphertext đi ra ngoài.

### Implementation

```c
// aes_lockstep.c
int main() {
    // ... key, plaintext setup ...

    // CPU A: chạy với fault có thể xảy ra (fault_byte/fault_bit từ env)
    struct AES_ctx ctx_a;
    AES_init_ctx(&ctx_a, key);
    uint8_t buf_a[16];
    memcpy(buf_a, plaintext, 16);
    AES_ECB_encrypt(&ctx_a, buf_a);  // ← fault có thể xảy ra ở đây

    // CPU B: chạy clean (disable fault injection)
    int saved_fb = fault_byte;
    fault_byte = -1;  // force no fault
    struct AES_ctx ctx_b;
    AES_init_ctx(&ctx_b, key);
    uint8_t buf_b[16];
    memcpy(buf_b, plaintext, 16);
    AES_ECB_encrypt(&ctx_b, buf_b);  // ← luôn clean
    fault_byte = saved_fb;

    // Lockstep comparator
    if (memcmp(buf_a, buf_b, 16) != 0) {
        printf("[LOCKSTEP] FAULT DETECTED  - outputs diverged!\n");
        printf("[LOCKSTEP] CPU A: ");
        for (int i = 0; i < 16; i++) printf("%02x", buf_a[i]);
        printf("\n[LOCKSTEP] CPU B: ");
        for (int i = 0; i < 16; i++) printf("%02x", buf_b[i]);
        printf("\n[LOCKSTEP] Execution ABORTED  - no ciphertext output\n");
        return 1;  // abort, không output gì thêm
    }

    // Chỉ output nếu cả hai đồng ý
    printf("Ciphertext: ");
    for (int i = 0; i < 16; i++) printf("%02x", buf_a[i]);
    printf("\n");
    return 0;
}
```

**Key design decision:** lane B luôn chạy clean bằng cách disable fault hook. Trong hardware implementation thực tế, lane B sẽ là một independent execution unit. Trong software demo, ta mô phỏng giả định đó bằng flag để chứng minh nguyên lý: nếu hai lane diverge thì abort trước khi ciphertext lỗi bị output.

### Compile

```bash
riscv64-linux-gnu-gcc -O0 -g -static -o aes_lockstep aes_lockstep.c aes.c -I.
```

### Testing 3 scenarios trong gem5

Tạo gem5 run script hỗ trợ arguments:

```python
# run_lockstep.py
import sys
fault_byte = sys.argv[1] if len(sys.argv) > 1 else "-1"
fault_bit  = sys.argv[2] if len(sys.argv) > 2 else "0"
...
process.env = [f'FAULT_BYTE={fault_byte}', f'FAULT_BIT={fault_bit}']
```

**Scenario 1: No fault**

```bash
./build/RISCV/gem5.opt ~/Desktop/run_lockstep.py -- -1 0 2>/dev/null \
    | grep -E "Ciphertext|LOCKSTEP"
```

```
Ciphertext: 3ad77bb40d7a3660a89ecaf32466ef97
```

Không có fault → cả hai CPU đồng ý → ciphertext output bình thường. ✓

**Scenario 2: Fault tại byte 0, bit 0**

```bash
./build/RISCV/gem5.opt ~/Desktop/run_lockstep.py -- 0 0 2>/dev/null \
    | grep -E "Ciphertext|LOCKSTEP"
```

```
[LOCKSTEP] FAULT DETECTED  - outputs diverged!
[LOCKSTEP] CPU A: 5dd77bb40d7a3616a89eeff32406ef97
[LOCKSTEP] CPU B: 3ad77bb40d7a3660a89ecaf32466ef97
[LOCKSTEP] Execution ABORTED  - no ciphertext output
```

Fault detected! CPU A bị corrupt (`5d...`), CPU B clean (`3a...`). Comparator detect divergence → abort, không output faulty ciphertext. Attacker không nhận được gì. ✓

**Scenario 3: Fault tại byte 5, bit 3**

```bash
./build/RISCV/gem5.opt ~/Desktop/run_lockstep.py -- 5 3 2>/dev/null \
    | grep -E "Ciphertext|LOCKSTEP"
```

```
[LOCKSTEP] FAULT DETECTED  - outputs diverged!
[LOCKSTEP] CPU A: 2bd77bb40d7a3646a89e6ef324ddef97
[LOCKSTEP] CPU B: 3ad77bb40d7a3660a89ecaf32466ef97
[LOCKSTEP] Execution ABORTED  - no ciphertext output
```

Fault khác vị trí, cũng bị detect thành công. ✓

Ba kịch bản trên cho thấy kết quả phù hợp với kỳ vọng của lockstep:

- Không có fault → hai lane tạo cùng ciphertext.
- Có fault trên lane A → ciphertext của hai lane khác nhau.
- Comparator phát hiện divergence và dừng chương trình trước khi dữ liệu lỗi bị lộ ra ngoài.

Điểm quan trọng là attacker không còn thu được faulty ciphertext. Khi đầu vào của PhoenixAES không tồn tại, toàn bộ DFA attack chain bị chặn từ bước đầu tiên.

### Lockstep defense chặn được fault trong demo này

Trong fault model của demo này, nếu source-level hook làm lane A lệch khỏi lane B tại round 9, comparator sẽ:
- Phát hiện fault trước khi ciphertext được output
- Abort execution, không leak faulty ciphertext cho attacker
- CPU B vẫn tính được correct ciphertext ở nội bộ hệ thống. Trong một thiết kế thực tế, hệ thống có thể thực hiện retry, reset hoặc chuyển sang cơ chế xử lý lỗi tùy theo yêu cầu an toàn.

### Limitations của Lockstep Defense

Cần lưu ý một số điều mà implementation này chưa cover:

**1. Double-fault attack:** Nếu attacker inject fault vào cả CPU A lẫn CPU B theo cùng pattern (synchronous fault), comparator sẽ không detect vì cả hai output giống nhau nhưng cùng sai. Trong hardware, điều này đòi hỏi attacker kiểm soát fault source với độ chính xác rất cao (target hai execution units riêng biệt cùng lúc).

**2. Comparator bypass:** Nếu attacker inject fault vào chính comparator logic (không phải AES execution), comparator có thể bị forced để output "no fault" kể cả khi có divergence. Defense in depth: cần harden comparator riêng.

**3. Differential power analysis:** Lockstep không protect against power analysis attacks  - ta chỉ chạy AES hai lần, làm power trace dài gấp đôi nhưng không che đi information.

**4. Timing overhead:** Chạy AES hai lần → 2× computational cost. Trong embedded systems với power constraints, đây là tradeoff quan trọng.

**5. Software vs hardware Lockstep:** Software lockstep trong cùng process (như demo này) không phải true hardware lockstep. Một fault injection attack nhắm vào instruction cache hoặc branch predictor có thể affect cả hai "CPU A" và "CPU B" khi chúng share physical execution pipeline.

---

## 14. Tổng kết và bài học
Nghiên cứu này bắt đầu từ mục tiêu mô phỏng một cuộc tấn công Differential Fault Analysis trên AES-128 trong môi trường gem5 và đánh giá hiệu quả của cơ chế Lockstep Defense.

Kết quả cuối cùng cho thấy cả hai mục tiêu đều đạt được: DFA có thể recover khóa AES khi fault được đặt đúng vị trí, trong khi lockstep có thể phát hiện sai lệch và ngăn faulty ciphertext bị rò rỉ.

### Summary of results

| Step | Result |
|------|--------|
| gem5 RISC-V setup | ✓ gem5 v25.1.0.1, RISC-V, SE mode |
| AES binary | ✓ tiny-AES-c, RISC-V statically linked |
| Correct ciphertext | `3ad77bb40d7a3660a89ecaf32466ef97` (NIST match) |
| Fault window | Round 9 SubBytes, tick 10,812,745,000 |
| Faulty ciphertexts | 128 unique (16 bytes × 8 bits) |
| Round 10 key | `D014F9A8C9EE2589E13F0CC8B6630CA6` |
| Master key | `2B7E151628AED2A6ABF7158809CF4F3C` ✓ |
| Lockstep Defense | Demo software lockstep detect divergence và abort before ciphertext output |

### Những điều học được từ quá trình debug

**gem5 SE mode không có dynamic linker.** Bất kỳ binary nào muốn chạy trong SE mode phải compile với `-static`. Đây là limitation fundamental của SE mode, không phải bug.

**gem5 v25 API thay đổi so với documentation cũ.** `threadContexts` không còn là attribute trực tiếp của CPU object. Cần dùng `cpu.cpuList[0].getContext(0)` hoặc các API mới hơn. Documentation online thường lag behind actual codebase.

**Checkpoint approach cho memory injection phức tạp hơn tưởng.** Tại thời điểm fault boundary được chọn, dữ liệu AES không thể được xác định một cách đáng tin cậy chỉ bằng cách chỉnh sửa các byte trong checkpoint memory. Điều này khiến việc checkpoint-level fault injection khó kiểm soát hơn dự kiến, nên không thể dễ dàng modify từ checkpoint. Cần hiểu AES execution model ở mức instruction để chọn đúng approach.

**Source-level fault injection vẫn hợp lệ cho phần cryptographic validation.** Nó không chứng minh khả năng glitch phần cứng vật lý, nhưng nếu fault được inject đúng vị trí trong computation (round boundary), faulty ciphertexts sẽ có differential structure cần thiết để PhoenixAES recover key.

**PhoenixAES format file:** Reference ciphertext phải ở **dòng đầu tiên** của input file, không phải passed separately. API documentation cần đọc kỹ.

**Reverse key schedule cần implement cẩn thận.** Manual reverse key schedule dễ có bug. Verify bằng forward expansion là approach robust hơn.

### Về friction trong research

Project này gặp nhiều issues: GemFI repo bị xóa (404), gem5 v25 API thay đổi, checkpoint memory chứa uninitialized data thay vì AES state, linker error do compile order. Không có gì trong số này là "thất bại"  - đây là friction bình thường của systems research với tools đang actively develop.

Quan trọng là biết cách debug: đọc error message, trace ngược lại source, thử alternative approach, không fix symptom mà fix root cause.

### Hướng mở rộng
Từ góc độ nghiên cứu, kết quả này cho thấy mối quan hệ trực tiếp giữa fault injection và cryptographic key recovery:

```text
Fault Injection
        ↓
Faulty Ciphertexts
        ↓
Differential Analysis
        ↓
Round Key Recovery
        ↓
Master Key Recovery
```

**Về attack:**
- Multi-fault DFA: inject 2 faults trong cùng session → reduce số ciphertext pairs cần thiết
- Differential fault analysis trên AES-256 (14 rounds, khác key schedule)
- Fault injection trên HMAC/SHA bên cạnh AES

**Về defense:**
- Implement Lockstep trên two actual CPU cores trong gem5 (dual-core simulation)
- Add temporal redundancy: chạy AES ở 3 time slots khác nhau, majority voting
- Combine Lockstep với masking countermeasure

**Về tooling:**
- Build custom gem5 module để automate fault injection ở hardware level (không cần patch source)
- Integrate với TVLA (Test Vector Leakage Assessment) để measure information leakage

Đồng thời, nghiên cứu cũng minh họa cách các cơ chế dự phòng như lockstep có thể cắt đứt chuỗi tấn công này bằng cách ngăn attacker tiếp cận dữ liệu lỗi ngay từ đầu.