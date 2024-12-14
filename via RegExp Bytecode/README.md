# Escaping V8 Sandbox via RegExp Bytecode (Chromium < 125.0.6422.60, V8 < 12.5.56)

## Setup

- Ubuntu 24.04.1 LTS (WSL)
- [0c1231b6414d19468d6f7a35ff5b6167626f57a5](https://chromium.googlesource.com/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5) (2024.03.22.)

[setup.zsh](./setup.zsh)

## Analysis

### RegExp bytecode

JavaScript의 [`RegExp`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/RegExp) object는 정규식 검사를 위해 사용되며, [`JSRegExp`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=38) class로 구현되어 있다.

![image](https://github.com/user-attachments/assets/30e126d5-9efc-4dc6-8c0c-d3d920eb73b3)

`JSRegExp` object가 생성되면 data array가 initialize된다. [`exec()`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec) method가 호출될 때 실행될 코드를 저장하는 [`code`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=70)와 [`bytecode`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=74) field는 최초에 [`kUninitializedValue`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=245)로 설정된다. 처음에는 `bytecode`를 compile하여 사용하다가, `exec()`이 일정 횟수 (tier-up ticks) 이상 호출되면 optimize된 `code`를 compile하여 사용한다. [`TierUpTick`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=131) field는 최초에 [`regexp_tier_up_ticks`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/flags/flag-definitions.h;l=2454)로 설정된다. `d8`의 실행 옵션에서 tier-up 활성화 여부와 tier-up ticks의 값을 조절할 수 있다.

![image](https://github.com/user-attachments/assets/2405e82f-ab08-4ffb-bc52-2b4445b6e519)

`exec()` method는 [`Runtime_RegExpExec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/runtime/runtime-regexp.cc;l=928)에서 처리하고, 내부적으로 [`RegExpExec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/runtime/runtime-regexp.cc;l=900) → [`RegExp::Exec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=323) → [`RegExpImpl::IrregexpExec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=749) → [`RegExpImpl::IrregexpPrepare()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=663) → [`RegExpImpl::EnsureCompiledIrregexp()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=444)로 들어간다.

`RegExpImpl::EnsureCompiledIrregexp()`에서는 `JSRegExp` object가 compile된 `code`나 `bytecode`를 가지고 있는지 검사한다. Initial compilation의 필요 여부를 의미하는 `needs_initial_compilation`은 data array의 `code` field의 값이 `kUninitializedValue`인 경우, 즉 한 번도 compile된 적이 없는 경우 `true`로 설정된다. Tier-up compilation의 필요 여부를 의미하는 `needs_tier_up_compilation`은 [`MarkedForTierUp()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.cc;l=214)의 return value가 `true`인 경우, 즉 [`regexp_tier_up`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/flags/flag-definitions.h;l=2450) flag의 값이 `true`이고 tier-up ticks가 0에 도달한 경우 `true`로 설정된다. 최종적으로 `needs_initial_compilation`과 `needs_tier_up_compilation` 중 하나라도 `true`인 경우 [`RegExpImpl::CompileIrregexp()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=547)를 호출한다.

`RegExpImpl::CompileIrregexp()`에서는 먼저 `compilation_target`을 설정하고 그에 맞게 compile을 진행한다. `compilation_target`은 [`ShouldProduceBytecode()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.cc;l=202)의 return value가 `true`인 경우 `RegExpCompilationTarget::kBytecode`로, 그렇지 않은 경우 `RegExpCompilationTarget::kNative`로 설정한다. `bytecode`를 compile할 때는 `code` field에 [`RegExpInterpreterTrampoline`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/builtins/builtins-definitions.h;l=899)을 넣고, tier-up이 진행되어 `code`를 compile할 때는 `bytecode` field를 `kUninitializedValue`로 초기화한다.

Tier-up이 진행되기 전까지는 `RegExpImpl::IrregexpExec()`에서 [`RegExpImpl::IrregexpExecRaw()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=679) → [`IrregexpInterpreter::MatchForCallFromRuntime()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=1149) → [`IrregexpInterpreter::Match()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=1058) → [`IrregexpInterpreter::MatchInternal()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=1073) → [`RawMatch()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=386)로 들어가서 `bytecode`를 decode하고 실행한다. Tier-up이 진행된 후에는 [`NativeRegExpMacroAssembler::Match()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-macro-assembler.cc;l=370) → [`NativeRegExpMacroAssembler::Execute()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-macro-assembler.cc;l=430)로 들어가서 compile된 `code`를 실행한다.

## Exploitation

### Idea

![image](https://github.com/user-attachments/assets/b5caca7d-fbf6-464e-88aa-d2f0af5aeef0)

Data array의 `bytecode` field에 저장된 array는 V8 sandbox 내부에 있기 때문에 arbitrary address write primitive가 있으면 bytecode를 임의로 수정할 수 있다. `RawMatch()`에서 `bytecode`를 실행할 때 사용하는 가상의 [`registers`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=445)와 [`backtrack_stack`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=447)이 모두 `RawMatch()`의 stack frame 내부에 있고, `registers`에 접근할 때 bounds check가 존재하지 않기 때문에 stack에서 `registers`의 범위를 벗어난 위치의 값을 읽거나 쓸 수 있다. 이를 이용하여 ROP가 가능하다.

### Get binary base address - Bypass PIE

![image](https://github.com/user-attachments/assets/d4baf73d-046b-41e9-ac89-dc1038a7b086)

Debug build 기준 register index 72, 73에 위치한 `RawMatch()`의 return address에 저장된 값을 가져와서 binary base address를 계산하면 `d8` binary에 있는 gadget들을 ROP에 사용할 수 있다.

### Get stack address - Bypass ASLR

![image](https://github.com/user-attachments/assets/316147b0-01d7-479c-aa8b-0479e4f8ecd0)

Debug build 기준 register index 70, 71에 위치한 `RawMatch()`의 stack frame pointer에 저장된 값을 가져오면 stack address를 ROP에 사용할 수 있다.

### Execute command via execve

임의의 command (e.g. `/bin/sh`)를 stack에서 ROP chain과 겹치지 않는 위치에 넣어 두고, 그 주소를 가져와서 `execve` system call의 인자로 전달하면 command를 실행할 수 있다.

[pwn.js](./pwn.js)

![image](https://github.com/user-attachments/assets/766373a0-bcec-4a52-9c43-0c8e62a7d732)

## Patch

> [[regex] Check bounds on register access in interpreter](https://chromium.googlesource.com/v8/v8/+/b9349d97fd44aec615307c9d00697152da95a66a) (2024.03.22.)
>
> Registers in the interpreter are stored on the stack/heap outside the sandbox. The register index is stored in the bytecode, which is (not yet) in trusted space.

## References

- [V8 Sandbox escape via regexp - Chromium Issues](https://issues.chromium.org/issues/330404819)
- [rycbar77/V8-Sandbox-Escape-via-Regexp - Github](https://github.com/rycbar77/V8-Sandbox-Escape-via-Regexp)
