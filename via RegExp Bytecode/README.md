# Escaping V8 Sandbox via RegExp Bytecode (Chromium < 125.0.6422.60, V8 < 12.5.56)

## Setup

- Ubuntu 24.04.1 LTS (WSL)
- [0c1231b6414d19468d6f7a35ff5b6167626f57a5](https://chromium.googlesource.com/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5) (2024.03.22.)

[setup.zsh](./setup.zsh)

## Analysis

### RegExp bytecode

JavaScript의 [`RegExp`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/RegExp) object는 regular expression에 관련된 작업들을 수행하며, [`JSRegExp`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=38) class로 구현되어 있다.

![image](https://github.com/user-attachments/assets/30e126d5-9efc-4dc6-8c0c-d3d920eb73b3)

`JSRegExp` object가 생성되면 data array가 initialize된다. [`exec()`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec) method가 호출될 때 실행될 코드를 저장하는 [code](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=70) (index: [`kIrregexpLatin1CodeIndex`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=187) `== 3`)와 [bytecode](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=74) (index: [`kIrregexpLatin1BytecodeIndex`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=190) `== 5`) field는 [`kUninitializedValue`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=245) (`== -1`)로 설정된다. Code의 tier-up이 진행되기 위해 필요한 실행 횟수를 의미하는 tier-up ticks (index: [`kIrregexpTicksUntilTierUpIndex`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.h;l=209) `== 10`) field는 [`regexp_tier_up_ticks`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/flags/flag-definitions.h;l=2454) (default: 1)로 설정된다. `d8`의 실행 옵션에서 tier-up의 활성화 여부와 tier-up ticks의 값을 조절할 수 있다.

![image](https://github.com/user-attachments/assets/2405e82f-ab08-4ffb-bc52-2b4445b6e519)

`exec()` method는 [`Runtime_RegExpExec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/runtime/runtime-regexp.cc;l=928)에서 처리하고, 내부적으로 [`RegExpExec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/runtime/runtime-regexp.cc;l=900) → [`RegExp::Exec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=323) → [`RegExpImpl::IrregexpExec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=749) → [`RegExpImpl::IrregexpPrepare()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=663) → [`RegExpImpl::EnsureCompiledIrregexp()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=444)로 들어간다.

`RegExpImpl::EnsureCompiledIrregexp()`에서는 `JSRegExp` object가 compile된 code를 가지고 있는지 검사한다. Initial compilation의 필요 여부를 의미하는 [`needs_initial_compilation`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=449)은, data array의 code field의 값이 `kUninitializedValue`인 경우, 즉 한 번도 compile된 적이 없는 경우 `true`로 설정된다. Tier-up compilation의 필요 여부를 의미하는 [`needs_tier_up_compilation`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=454)은, [`MarkedForTierUp()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/objects/js-regexp.cc;l=214)의 return value가 `true`인 경우, 즉 [`regexp_tier_up`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/flags/flag-definitions.h;l=2450) flag가 enable되어 있고 tier-up ticks가 0에 도달한 경우 `true`로 설정된다. 최종적으로 `needs_initial_compilation`과 `needs_tier_up_compilation` 중 하나라도 `true`인 경우 [`RegExpImpl::CompileIrregexp()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=547)를 호출한다.

`RegExpImpl::CompileIrregexp()`에서는 먼저 [`compilation_target`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=574)을 설정한다. Initial compilation이 필요한 경우 `RegExpCompilationTarget::kBytecode`로 설정하고, tier-up compilation이 필요한 경우 `RegExpCompilationTarget::kNative`로 설정한다. `compilation_target`이 `RegExpCompilationTarget::kBytecode`인 경우 bytecode field에 bytecode array를 저장하고 code field에는 trampoline을 저장한다. `compilation_target`이 `RegExpCompilationTarget::kNative`인 경우 code field에 compile된 code를 저장하고 bytecode field는 다시 `kUninitializedValue`로 설정한다.

Initial compilation이 완료되어 bytecode가 생성되었지만 tier-up은 아직 진행되지 않은 경우, [`RegExpImpl::IrregexpExec()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=749) → [`RegExpImpl::IrregexpExecRaw()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp.cc;l=679) → [`IrregexpInterpreter::MatchForCallFromRuntime()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=1149) → [`IrregexpInterpreter::Match()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=1058) → [`IrregexpInterpreter::MatchInternal()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=1073) → [`RawMatch()`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=386)로 들어가서 bytecode를 실행한다.

## Exploitation

### Idea

![image](https://github.com/user-attachments/assets/b5caca7d-fbf6-464e-88aa-d2f0af5aeef0)

Data array의 bytecode field에 저장된 array는 V8 sandbox 내부에 있기 때문에 arbitrary address write primitive가 있는 경우 bytecode를 임의로 수정할 수 있다. `RawMatch()`에서 bytecode를 실행할 때 사용하는 가상의 [`registers`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=445)와 [`backtrack_stack`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-interpreter.cc;l=447)이 모두 `RawMatch()`의 stack frame 내부에 위치하고, `registers`에 접근할 때 bounds check가 존재하지 않기 때문에, [`PUSH_REGISTER`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-bytecodes.h;l=38), [`POP_REGISTER`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-bytecodes.h;l=47), [`SET_REGISTER`](https://source.chromium.org/chromium/v8/v8/+/0c1231b6414d19468d6f7a35ff5b6167626f57a5:src/regexp/regexp-bytecodes.h;l=43) 등의 instruction들을 이용하여 stack에서 값을 읽거나 쓸 수 있고, 이를 이용하여 ROP가 가능하다.

### Get binary base - Bypass PIE

![image](https://github.com/user-attachments/assets/d4baf73d-046b-41e9-ac89-dc1038a7b086)

`RawMatch()`의 return address는 register index 72, 73에 위치한다. 이 값을 register index 0, 1로 가져와서 binary base를 계산하면 `d8` binary에 있는 gadget들을 ROP에 사용할 수 있다.

### Get stack address - Bypass ASLR

![image](https://github.com/user-attachments/assets/316147b0-01d7-479c-aa8b-0479e4f8ecd0)

`RawMatch()`의 `rbp`는 register index 70, 71에 위치한다. 이 값을 register index 2, 3으로 가져와서 stack address를 계산하면 stack에 저장된 임의의 값의 address를 reference할 수 있다.

### Execute command via execve

임의의 command (e.g. `/bin/sh`)를 stack에서 ROP chain과 겹치지 않는 임의의 위치에 쓰고, 그 address를 가져와서 `execve` system call의 인자로 전달하면 command를 실행할 수 있다.

[pwn.js](./pwn.js)

![image](https://github.com/user-attachments/assets/766373a0-bcec-4a52-9c43-0c8e62a7d732)

## Patch

> [[regex] Check bounds on register access in interpreter](https://chromium.googlesource.com/v8/v8/+/b9349d97fd44aec615307c9d00697152da95a66a) (2024.03.22.)
> Registers in the interpreter are stored on the stack/heap outside the sandbox. The register index is stored in the bytecode, which is (not yet) in trusted space.

```diff
diff --git a/src/regexp/regexp-interpreter.cc b/src/regexp/regexp-interpreter.cc
index 31fe503..13cf076 100644
--- a/src/regexp/regexp-interpreter.cc
+++ b/src/regexp/regexp-interpreter.cc
@@ -176,6 +176,7 @@
                        int output_register_count)
       : registers_(total_register_count),
         output_registers_(output_registers),
+        total_register_count_(total_register_count),
         output_register_count_(output_register_count) {
     // TODO(jgruber): Use int32_t consistently for registers. Currently, CSA
     // uses int32_t while runtime uses int.
@@ -188,10 +189,17 @@
     // Initialize the output register region to -1 signifying 'no match'.
     std::memset(registers_.data(), -1,
                 output_register_count * sizeof(RegisterT));
+    USE(total_register_count_);
   }
 
-  const RegisterT& operator[](size_t index) const { return registers_[index]; }
-  RegisterT& operator[](size_t index) { return registers_[index]; }
+  const RegisterT& operator[](size_t index) const {
+    SBXCHECK_LT(index, total_register_count_);
+    return registers_[index];
+  }
+  RegisterT& operator[](size_t index) {
+    SBXCHECK_LT(index, total_register_count_);
+    return registers_[index];
+  }
 
   void CopyToOutputRegisters() {
     MemCopy(output_registers_, registers_.data(),
@@ -202,6 +210,7 @@
   static constexpr int kStaticCapacity = 64;  // Arbitrary.
   base::SmallVector<RegisterT, kStaticCapacity> registers_;
   RegisterT* const output_registers_;
+  const int total_register_count_;
   const int output_register_count_;
 };
```

`RawMatch()`에서 `registers`에 접근할 때 bounds check가 추가되었다.

## References

- [V8 Sandbox escape via regexp - Chromium Issues](https://issues.chromium.org/issues/330404819)
- [rycbar77/V8-Sandbox-Escape-via-Regexp - Github](https://github.com/rycbar77/V8-Sandbox-Escape-via-Regexp)
