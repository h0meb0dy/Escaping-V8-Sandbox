# Escaping V8 Sandbox via Turbofan JIT Spraying (Chromium < 117.0.5938.62, V8 < 11.7.156)

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

![image](https://github.com/user-attachments/assets/8856cd93-8458-4dbb-a5a6-f7540faf4d11)

![image](https://github.com/user-attachments/assets/6102f7ed-dbce-4392-b44b-141dd6be7f5c)

![image](https://github.com/user-attachments/assets/e696a063-f7fc-4e99-ac19-125778996332)

![image](https://github.com/user-attachments/assets/22199a6d-cccd-419c-90b3-b1150c153db3)

![image](https://github.com/user-attachments/assets/2bebb2a1-065d-4e23-929d-b7f84c75e845)

Float number들로 구성된 array를 return하는 함수 `jit()`를 Turbofan으로 JIT compile하면 array의 element들이 코드에 raw number로 삽입된다. `Code` object는 함수를 호출할 때 실행되는 instruction의 시작 주소인 `instruction_start`를 8-byte full pointer로 가지고 있는데, 이 object는 V8 sandbox 내부에 위치하기 때문에 arbitrary address write primitive가 있으면 `instruction_start`를 조작하여 `rip`를 control할 수 있다. `rip`를 코드에 삽입된 raw number의 주소(e.g. `0x601965b040ac`)로 옮기면 해당 raw number가 8바이트 길이의 shellcode로 동작하게 된다. 

### Constructing shellcode chain

8바이트 길이의 shellcode로는 원하는 코드를 실행할 수 없을 가능성이 높다. 삽입되는 raw number들 간의 거리는 `0xff`를 초과하지 않기 때문에 shellcode의 마지막 2바이트에 `jmp` instruction을 넣어서 다음 shellcode로 jump하도록 하여 shellcode chain을 구성할 수 있다.

[shellcode.py](./shellcode.py)

![image](https://github.com/user-attachments/assets/2069e7b1-bb65-4977-8c60-69f28da07944)

![image](https://github.com/user-attachments/assets/948b894f-5981-4725-888c-17e7bc21c53d)

만들어진 shellcode array에 위와 같이 중복되는 값이 있는 경우 register에 저장했다가 재사용하도록 compile되기 때문에 shellcode가 동작하지 않는다. 따라서 코드의 순서를 바꾸거나 `nop`의 위치를 옮겨서 중복되는 값이 없도록 만들어야 한다.

[pwn.js](./pwn.js)

![image](https://github.com/user-attachments/assets/63213bef-e95b-416e-a299-71ad2f022dd9)

## Patch

> [[sandbox] Enable code pointer sandboxing](https://chromium.googlesource.com/v8/v8/+/c8d039b05081b474ef751411a5c76ca01900e49a) (2023.07.11.)
>
> [Revert "[sandbox] Enable code pointer sandboxing"](https://chromium.googlesource.com/v8/v8/+/bc795ebd90a5a7c957b644da5fac369eb88aa87a) (2023.07.11.)
>
> [Reland "[sandbox] Enable code pointer sandboxing"](https://chromium.googlesource.com/v8/v8/+/7df23d5163a10a12e4b4262dd4e78cfb7ec97be0) (2023.07.11.)
