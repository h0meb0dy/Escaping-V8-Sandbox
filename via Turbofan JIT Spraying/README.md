# Escaping V8 Sandbox via Turbofan JIT spraying (Chromium < 117.0.5938.62, V8 < 11.7.156)

## Setup

- Ubuntu 22.04.5 LTS (WSL)
- [4512c6eb7189c21f39420ddf8d9ff4f05a4a39b4](https://chromium.googlesource.com/v8/v8/+/4512c6eb7189c21f39420ddf8d9ff4f05a4a39b4) (2023.07.11.)

[setup.zsh](./setup.zsh)

## Exploitation

### JIT (just-in-time) spraying

```js
function jit() { return [1.1, 2.2, 3.3]; }

// compile jit() with turbofan
for (let i = 0; i < 0x10000; i++) { jit(); jit(); }

% DebugPrint(jit);
```

![image](https://github.com/user-attachments/assets/d1c0bfd2-5537-4a13-950c-67c500bb8476)

![image](https://github.com/user-attachments/assets/f8bbc6e0-ebac-4e8b-b2e3-08eef9b7dff2)

![image](https://github.com/user-attachments/assets/01cd4fb5-0653-43f0-936d-211103eb47a2)

![image](https://github.com/user-attachments/assets/d85c591d-c207-4b80-849e-3a89dfc41714)

![image](https://github.com/user-attachments/assets/d23d985a-3a81-4807-84c7-66b459f39105)

Float number들로 구성된 array를 return하는 함수 `jit()`를 Turbofan으로 JIT compile하면 array의 element들이 코드에 raw number로 삽입된다.

`Code` object는 함수를 호출할 때 실행되는 instruction의 시작 주소인 `instruction_start`를 8-byte full pointer로 가지고 있는데, 이 object는 V8 sandbox 내부에 위치하기 때문에 arbitrary address write primitive를 가지고 있는 경우 `instruction_start`를 overwrite하여 `rip`를 control할 수 있다. `rip`를 코드에 삽입된 raw number의 주소(e.g. `0x5637e00040ac`)로 옮기면 해당 raw number가 8바이트 길이의 shellcode로 동작하게 된다.

### Constructing shellcode chain

Shellcode를 실행할 수 있긴 하지만, 8바이트로는 원하는 코드를 실행할 수 없을 가능성이 높다. 삽입되는 raw number들 간의 길이가 1바이트를 넘어가지 않기 때문에 shellcode의 마지막 2바이트에 `jmp` instruction을 넣어서 다음 shellcode로 jump하도록 하여 shellcode chain을 구성할 수 있다. 따라서 shellcode를 작성할 때 한 개의 segment가 6바이트를 넘어가지 않도록 해야 한다.

[shellcode.py](./shellcode.py)

![image](https://github.com/user-attachments/assets/e46d00aa-3765-4d7e-a1b1-2319e8a7dc07)

![image](https://github.com/user-attachments/assets/ecbfa022-f3df-4878-b274-4a2ce1885335)

만들어진 shellcode에 중복되는 값이 있을 경우, register에 저장했다가 재사용하도록 compile되기 때문에 shellcode가 동작하지 않는다. 따라서 코드의 순서를 바꾸거나 `nop`의 위치를 옮겨서 중복되는 값이 없도록 만들어야 한다. 

[pwn.js](./pwn.js)

![image](https://github.com/user-attachments/assets/cfe4b984-4400-4732-a7db-6e611a7de787)

## Patch

> [sandbox - First step towards sandbox CFI](https://chromium.googlesource.com/v8/v8/+/ee48926106051afb784d8f39c31aab0d2a04823f) (2023.06.09.)
>
> This CL implements very basic code pointer sandboxing to ensure that indirect control-flow transfers inside the sandbox always land on a valid code entrypoint. This is achieved by setting up a dedicated code pointer table (CPT) that contains the code entrypoints and then using CodePointerHandles (i.e., table indices) to reference these entries from inside the sandbox. This is essentially the same mechanism used for the external pointer table (EPT), except that the CPT doesn't use type tags.

> [sandbox - Enable code pointer sandboxing](https://chromium.googlesource.com/v8/v8/+/c8d039b05081b474ef751411a5c76ca01900e49a) (2023.07.11.)
> [Revert "sandbox - Enable code pointer sandboxing"](https://chromium.googlesource.com/v8/v8/+/bc795ebd90a5a7c957b644da5fac369eb88aa87a) (2023.07.11.)
> [Reland "sandbox - Enable code pointer sandboxing"](https://chromium.googlesource.com/v8/v8/+/7df23d5163a10a12e4b4262dd4e78cfb7ec97be0) (2023.07.11.)
