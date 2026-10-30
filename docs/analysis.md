# Analysis

## 1. Host application

SolaraV3.exe is Rust with Tauri 2, not C++. The build path
`C:\Users\quivi\.cargo\registry\...\tauri-2.x` and the crate name
`solara_v3_lib::AppState` are both present. The startup routine at RVA 0x10010
runs a fixed sequence:

```
setup: data dirs created
setup: writing startup asset
setup: startup asset ready
setup: no migration pending, loading DLL
```

The DLL loader at RVA 0x5400 seeds and starts the native engine, then reads the
return value of Start as the bridge port.

```
copy_dll: canonical DLL missing, seeding from bundled resource
dll load: preparing SolaraV3.dll
dll load: Start() returned bridge port
```

## 2. Bridge

One local HTTP server on 127.0.0.1, built on crow. All four endpoints live in a
single 4800 byte function at 0x180215e90, confirmed by scanning for rip relative
loads of the endpoint strings.

| Endpoint | String VA |
|---|---|
| /state | 0x180f49d60 |
| /execute_script | 0x180f49d68 |
| /request | 0x180f49d78 |
| /client_manager | 0x180f49d88 |

Start is the only export, at 0xAECAC0, 21909 bytes.

## 3. Script engine

The DLL carries a full Luau compiler. RTTI shows 99 entries under the Luau
namespace and 87 of them are compiler visitors: BuiltinVisitor, ConstUpvalueVisitor,
CostVisitor, ShapeVisitor, ReturnVisitor and others, plus the whole AstExpr family.
Runtime strings confirm the surface: getgenv, getscriptbytecode, upvalue and
closure limit messages.

Flow is source to Luau::Compile inside the DLL to bytecode, then marshalled into
the structures written into Roblox.

## 4. Memory patching

OpenProcess appears three times in 0x1801ad0f0 with access 0x1FFFFE, 0x438 and
0x1038. The PID comes from a field in the client struct at offset 0x31.

| Step | Function | Action |
|---|---|---|
| 1 | 0x1801d0cf0 | VirtualAllocEx 0x40 bytes |
| 2 | 0x180af2be0 | VirtualAllocEx len+0x21, write header and string |
| 3 | 0x1801d0cf0 | write the 64 byte parameter block |
| 4 | 0x180245890 | patch a single 8 byte pointer |
| 5 | 0x1801d6280 | read back allocations, write 0x50 byte control block |

Every allocation uses r9d = 0x3000 (MEM_COMMIT or MEM_RESERVE) and passes 4 as
the fifth argument, which is PAGE_READWRITE. The fifth argument is visible on the
stack:

```
1801d6301: mov dword ptr [rsp + 0x20], 4
1801d630b: mov r9d, 0x3000
1801d6311: mov r8d, 0x50
```

Read and write memory, never executable. No PAGE_EXECUTE_READWRITE on any of
these paths.

The patch cluster sits in 0x180b21900, 11496 bytes. It walks the target with
VirtualQueryEx and flips single bytes.

```c
VirtualQueryEx(proc, 0, &mbi);
WriteProcessMemory(proc, target, "\x90", 1);
ReadProcessMemory(proc, addr, &byte, 1);
WriteProcessMemory(proc, addr, &zero, 1);
```

No allocation in that function. It patches code that is already there.

## 5. What is not there

Zero hits across the whole binary for CreateRemoteThread, NtCreateThreadEx,
RtlCreateUserThread, SetThreadContext, SuspendThread, ResumeThread and
ZwCreateThread.

QueueUserAPC is imported and called from one wrapper at 0x18022fa40. That wrapper
has two callers and both are destructors:

```
0x22fa40  <-  0x215ce0, 0x2364f0
```

The destructor body drains state, calls the wrapper, closes a handle and frees the
object. The APC target is _guard_check_icall, a stock MSVC CFG thunk. That is a
thread teardown pattern, not injection.

Also zero hits for ScriptContext, DataModel, TaskScheduler and
WaitingHybridScriptsJob in an 18 MB DLL. A name based executor would carry those
strings. This one does not, so it works on offsets.

## 6. Assets

bin/ALGA is 32 decoded bytes. bin/DLCTBL_FRTS is 122848 bytes at entropy 7.98.

The structure is the useful part. The first 240 bytes are byte identical to the
copy embedded in SolaraV3.exe, and they diverge exactly at byte 240, which is a
16 byte boundary. Static header, per build body, 122608 body bytes, block aligned.

Keys tried: ALGA, SHA256 of ALGA, SHA256 of the header, first 32 header bytes,
HKDF and PBKDF2 derivations. Modes tried: AES CBC, CFB, OFB, CTR in both counter
orders, ChaCha20 with 8 and 12 byte nonces.

All failed, maximum printable ratio 0.40 which is noise. The decisive test was
AEAD tag verification for AES 256 GCM and ChaCha20 Poly1305. Every key and nonce
combination was rejected. Tags do not produce false positives.

Reason: the crypto in the host belongs to rustls and ring, used for HTTPS. Its
imports are ProcessPrng, BCryptGenRandom and eleven Cert functions. In the DLL the
single CryptDecrypt call site at 0x180c662d4 has no callers at all.

The key is not in either file. Static analysis ends here.
