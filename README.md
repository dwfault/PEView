## The bug

A bug exists in PEView.exe mentioned in the book ReverseCore(http://www.reversecore.com/111) and results in memory corruption and denail of service. The memory corruption vulnerability could be used as information disclosure.

PEView is a tool that helps identify the structure of PE files. And the crash happens when parsing `Time Date Stamp`in PE file structure. Its update was stopped and a `patched` version was provided in this repository.



## Crash log

(d44.8b4): Access violation - code c0000005 (!!! second chance !!!)

eax=00000008 ebx=01420000 ecx=0000000b edx=30303030 `esi=3240aa50` edi=0012f3fb

eip=00404788 esp=0012f05c ebp=0012f464 iopl=0         nv up ei pl nz ac pe nc

cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010216

PEview+0x4788:

00404788 f3a4            `rep movs byte ptr es:[edi],byte ptr [esi]`

0:000> kb

ChildEBP RetAddr  Args to Child              

WARNING: Stack unwind information not available. Following frames may be wrong.

0012f464 0040400b 0012f4cc 940118c8 00000000 PEview+0x4788

0012f4dc 7647c4e7 003e0406 0000000f 00000000 PEview+0x400b

00000000 00000000 00000000 00000000 00000000 USER32!InternalCallWinProc+0x23



## Analysis

The bug is because of abusing of Window API GetDateFormat.

code:00404A81                 push    edi

code:00404A82                 lea     eax, [ebp+SystemTime]

code:00404A85                 push    10h             ; cchDate

code:00404A87                 push    edi             ; lpDateStr

code:00404A88                 push    offset Format   ; "yyyy/MM/dd ddd "

code:00404A8D                 push    eax             ; lpDate

code:00404A8E                 push    0               ; dwFlags

code:00404A90                 push    800h            ; Locale

code:00404A95                 call    ds:GetDateFormatA

code:00404A9B                 lea     edi, [edi+eax-1]

code:00404A9F                 lea     eax, [ebp+SystemTime]

code:00404AA2                 push    0Dh             ; cchTime

code:00404AA4                 push    edi             ; lpTimeStr

code:00404AA5                 push    offset aHhMmSsUtc ; "HH:mm:ss UTC"

code:00404AAA                 push    eax             ; lpTime

code:00404AAB                 push    0               ; dwFlags

code:00404AAD                 push    800h            ; Locale

code:00404AB2                 call    ds:GetTimeFormatA ;

- In the global data area of process, there exists a pointer and a char buffer as follows to strore date and time information.

  `40aa4c` 40aa50

  `40aa50` ...

  They are adjacent and the pointer point to the buffer. The buffer is used to store the date and time information so it was passed into the API GetDateFormat as lpDateStr.

- At RVA 00404A9B, the return value of GetDateFormat is not checked but directly in used. 

  - In coutries where English is the primary language nothing happens, because the cchDate just fix the buffer.
  
  - In coutries like China, the ccDate is just off-by-one smaller than the output `yyyy/MM/dd ddd ` because of wide chars. The API returns 0. This zero is stored in eax, and lpTimeStr for GetTimeFormat values 0x40aa4f(`0x40aa50-0x1` by `lea edi, [edi+eax-1]`). Thus when GetTimeFormat was called, the pointer of char buffer was changed from `40aa50` to `3240aa50`. The time was 23:10:49, the `32` refers to `2` and is the LSB of this DWORD. The `3240aa50` is an invalid pointer for most situations.

- If some module was loaded in VA `30000000`to `32000000`, it was possible to be a valid memory address. When display in the window, the content of the memory should be displayed thus cause information disclosure.


## The Patch

The patch modified the module in one single byte by increasing the value of cchDate to some proper extend and fixed this problem.
