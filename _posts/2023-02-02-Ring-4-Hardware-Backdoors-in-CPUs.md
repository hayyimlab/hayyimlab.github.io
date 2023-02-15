---
title: "Ring -4?: Hardware Backdoors in CPUs"
date: 2023-02-02 13:00:00 +0900
author: "Donghyun"
tags: [research, rosenbridge, sandsifter, nightshyft, hardware backdoor, negative ring, god mode]
---

# Intro

지난여름, 하임시큐리티 내부 세미나에서 박서빈(moonoik) 선임 연구원님이 한때 뜨거운 감자였던 [프로세서 취약점](https://meltdownattack.com/)에 대한 대응으로 제시된 당시 Intel, AMD 등의 microcode 보안 업데이트를 필두로, 하드웨어 백도어에 대한 내용을 발표했습니다. 발표를 들었을 때도 크게 흥미를 느꼈지만 회사 일이 한창 바쁠 때라 우선순위에서 밀렸었는데, 연초에 개인 연구 시간이 일부 주어져 조금은 여유롭게 관련 연구를 살펴보았습니다.

하드웨어 백도어에는 다양한 방법론이 적용될 수 있습니다. Intel Management Engine(ME), AMD Platform Security Processor(PSP) 등과 관련한 방법론도 있고, 2018년 중국 스파이가 미국 기술 공급망에 침투해 Supermicro 사의 마더보드에 물리적인 칩을 심은 것[^1]도 하드웨어 백도어의 일례입니다.

![The Big Hack](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/The Big Hack.jpg){: width="500" }

여러 보안 컨퍼런스에 거의 매년 발표 주제로 등장할 정도로 지속해서 연구되는 분야이고, 최근에는 chip-red-pill 팀에 의해 [Intel microcode decryptor](https://github.com/chip-red-pill/MicrocodeDecryptor)가 개발되어 이목을 끌었습니다.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Today we&#39;ve published Intel Microcode decryptor! It gives you an amazing opportunity for researching x86 platforms. You can understand how Intel mitigated spectre vulnerability, explore the implementation of Intel TXT, SGX,VT-x technologies! Enjoy it! <a href="https://t.co/CrMYbrPu03">https://t.co/CrMYbrPu03</a> <a href="https://t.co/pW6iQoUGLJ">pic.twitter.com/pW6iQoUGLJ</a></p>&mdash; Maxim Goryachy (@h0t_max) <a href="https://twitter.com/h0t_max/status/1549155542786080774?ref_src=twsrc%5Etfw">July 18, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

파고드니 흥미로운 것이 많았지만, 그만큼 양도 방대했기에 본 글에서는 인상 깊었던 연구 중 하나를 골라 얘기해보고자 합니다.

# project:rosenbridge

*project:rosenbridge*는 [Christopher Domas(xoreaxeaxeax)](https://www.linkedin.com/in/christopher-domas-39b3a5102/)가 Black Hat USA 2018을 통해 공개한 연구입니다.[^2][^3] 연구가 시사하는 바와 연구에 활용한 접근법, 방법론이 재밌고 참신하다고 생각되는 점이 여럿 있어서(특허를 리버싱 한다거나...) 글의 주제로 선정하였습니다. 글을 읽기 전 그의 [발표](https://youtu.be/_eSAF_qT_FY)를 가볍게 들어본다면 이해에 도움이 될 것입니다.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">GOD MODE UNLOCKED: hardware backdoors in some x86 CPUs<a href="https://t.co/Ph0IAL0Pyw">https://t.co/Ph0IAL0Pyw</a><br>White paper coming tomorrow. <a href="https://twitter.com/BlackHatEvents?ref_src=twsrc%5Etfw">@BlackHatEvents</a> <a href="https://t.co/qhZ1vFI7pL">pic.twitter.com/qhZ1vFI7pL</a></p>&mdash; domas (@xoreaxeaxeax) <a href="https://twitter.com/xoreaxeaxeax/status/1027642170860163072?ref_src=twsrc%5Etfw">August 9, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## Overview

본 연구에서 proof-of-concept으로 구현한 백도어는 ring 3(userland)에서 ring 0(kernel)로의 권한 상승을 제공하여 arbitrary unprivileged code에서 커널에 대한 제한 없는 접근을 가능케 합니다. 이때 백도어는 하드웨어 및 소프트웨어 커널 보안 메커니즘에 대한 수십 년의 진전을 무효화하는데, antivirus, address space protection, data execution prevention, code signing, control flow integrity, kernel integrity check 등의 보호 기법이 모두 백도어를 통해 우회됩니다.

백도어는 프로세서 제작 단계 또는 부팅 단계에서 구성된 processor configuration bit를 통해 활성화 또는 비활성화할 수 있으며, 일부 플랫폼에서는 기본적으로 활성화되어 있습니다(= open door).

*백서에서는 8086 architecture에서 파생된 프로세서 설계를 광범위하게 지칭하기 위해 "x86"이라는 용어를 사용합니다. 여기에는 32-bit와 64-bit 버전이 모두 포함됩니다.

## Target

연구는 x86 프로세서의 VIA C3 제품군을 대상으로 합니다.

연구 대상은 일부 x86 기술에 대해 출원된 특허에서 파생된 정보를 기반으로 선정하였습니다. [US8341419](https://patents.google.com/patent/US8341419)에 다음과 같은 내용이 있습니다:

> "일부 internal control register를 통해 사용자가 보안 메커니즘을 우회할 수 있다. 가령, ring 3에서 ring 0에 접근할 수 있다. 또한 이러한 control register들은 프로세서 설계자가 독점적으로 유지하려는 정보를 드러낼 수 있다. 이러한 이유로 많은 x86 프로세서 제조업체는 일부 control MSR(Model Specific Register)의 주소나 기능에 대한 어떠한 설명도 공개적으로 문서화하지 않았다."

위 특허의 소유자(VIA Technologies, Inc.) 및 출원 연도를 기준으로, VIA C3 프로세서가 연구 대상으로 선정되었습니다. Intel 및 AMD에서는 일반적으로 프로세서에 대한 중요한 통찰을 엿볼 수 있는 개발자 매뉴얼을 제공하지만 VIA 개발자 매뉴얼은 찾을 수 없었기에 본 연구는 상당한 시행착오가 수반되었습니다. 연구는 주로 Nehemiah core에서 테스트 되었지만 모든 VIA C3 프로세서에 적용할 수 있을 것으로 사료됩니다.

대상 프로세서가 현대 컴퓨터에는 더 이상 사용되지 않지만, 연구에서 제시된 보안 문제는 산업 전반에 걸쳐 매우 현실적인 관심사로 남아 있으며, 현대 시스템에 대한 프로세서 보안 연구에서 최첨단 기술을 획기적으로 발전시키기 위한 귀중한 사례 연구로 본 연구를 제안합니다.

## Backdoor Architecture

이전 섹션에서 논의된 바와 같이 "Apparatus and method for limiting access to model specific registers in a microprocessor"라는 제목의 [특허](https://patents.google.com/patent/US20100235645)는 일반적으로 프로세서 백도어로 이해되는 것의 존재를 강력하게 암시합니다. 이를 탐색하기 위해 다른 x86 특허를 조사하며 정보 조각을 모았습니다. (US8880851, US9043580, US9141389, US9146742, US9292470, US9317301, ...)

여러 정보를 기반으로, VIA가 non-x86 core를 C3 x86 CPU에 내장하고 있으며 이 내장 core는 특수 명령을 통해 활성화할 수 있고, 이를 통해 프로세서 보안 메커니즘을 우회할 수 있다고 결론 내렸습니다. 이는 비교적 잘 알려진 Intel ME와 AMD PSP를 어렴풋이 연상시키지만, VIA의 내장 core는 x86 core와 훨씬 더 밀접하게 결합하여 있는 것처럼 보였고, 이러한 기능에 대한 공개 문서를 찾을 수 없었기에 ME나 PSP보다 더 숨겨져 있는 다른 무언가로 비춰졌습니다. 이 때문에, 연구에서는 이 non-x86 core를 *deeply embedded core(DEC)*로 이름 붙였습니다.

[US8880851](https://patents.google.com/patent/US8880851)을 통해 *DEC*가 완전히 별개의 core가 아니라 pipeline 및 기타 architecture의 상당 부분을 x86 core와 공유한다고 추측하였습니다.

![US8880851-fig1](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/US8880851-fig1.png){: width="600" }

다른 여러 특허[^4]는 *DEC*가 execution pipeline의 일부 구성 요소를 x86 core와 공유하는 RISC 프로세서임을 시사하며, fetch phase 이후 pipeline이 분기될 가능성을 나타냅니다. RISC와 x86 core 간에 부분적으로 공유하는 register file에 대한 존재도 추측할 수 있습니다.[^5]

![US9043580-fig](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/US9043580-fig.png){: width="600" }

US8880851에 따르면 프로세서는 RISC core를 활성화하기 위해 MSR로 x86 core에 노출된 *global configuration register*를 사용합니다. RISC core가 활성화되면 RISC instruction sequence는 x86 instruction set에 추가된 새로운 instruction인 x86 *launch instruction으*로 시작됩니다.

Integrated execution pipeline과 shared register file을 포함한 *DEC*의 설계는 Intel ME나 AMD PSP와 같은 coprocessor보다 더 은밀하고 강력합니다. Protected memory를 수정할 수 있는 coprocessor들은 kernel, hypervisor, System Management Mode의 능력을 능가하는 'ring -3' layer of privilege로 불려왔습니다. 이에 본 연구는 *DEC*가 지금까지 발견된 가장 깊은 layer인 일종의 'ring -4'로 작용한다고 제안합니다.

![IA negative rings](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/IA negative rings.png){: width="600" }

*DEC*에 대한 가정이 맞는다면, *DEC*는 프로세서에서 일종의 백도어로 사용되어 가장 중요한 프로세서 보안 검사를 모두 은밀하게 우회할 수 있습니다; 우리는 이것을 *rosenbridge* backdoor라고 부릅니다.

## Register Analysis

x86의 MSR은 64-bit control register입니다. 디버깅, 성능 모니터링, 다양한 프로세서 기능 전환 등 많은 곳에 사용됩니다. MSR은 ring 0에서만 접근할 수 있습니다. x86 general- 및 special-purpose register와 달리 MSR은 이름이 아니라 주소로 접근 가능합니다. 유효한 MSR 주소 범위는 `0` 에서 `0xffffffff`까지 입니다.

앞서 언급한 US8341419의 내용처럼 MSR의 많은 부분이 공개 문서에 존재하지 않는 것이 일반적입니다.

![IA32_EFER control MSR](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/IA32_EFER control MSR.png){: width="600" }

문서화되지 않은 bit는 단순히 구현되지 않고 향후 사용을 위해 예약되는 경우도 많습니다. 그러나 프로세서에 확연한 영향을 미치는 문서화되지 않은 bit를 찾는 것은 그리 드문 일이 아닙니다.

연구에서는 문서화되지 않은 임의의 MSR 주소에 대해 `rdmsr` 명령을 수행하여 `#GP(0)` exception이 발생한다면 해당 MSR이 구현되지 않은 것으로 추론하고, exception 없이 성공적으로 명령이 수행된다면 MSR이 존재하는 것으로 추론하는 방식을 사용하였습니다. 그리고 (불행히도) C3에 대해 이러한 MSR fault analysis를 수행한 결과, 문서화되지 않았지만 구현되어 있는 1300개의 MSR이 식별되었습니다. 분석하기에는 너무 많습니다.

분석을 수월하게 하도록 연구에서는 x86 MSR에 대한 side-channel attack을 수행합니다. `rdmsr` 실행 전후로 `rdtsc` 명령을 사용해 `rdmsr`의 액세스 시간을 측정합니다. 이를 모든 MSR에 대해 자동으로 수행하는 MSR timing analysis code를 별도의 프로젝트인 [*project:nightshyft*](https://github.com/xoreaxeaxeax/nightshyft)로 구현하였지만 현재 해당 GitHub directory가 삭제된 상태입니다.[^6]

![side-channel attack](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/side-channel attack.png){: width="600" }

각 MSR에 대한 microcode가 다르므로 기능적으로 다른 MSR은 일반적으로 액세스 시간이 다를 것입니다. 가령, thermal sensor MSR에 액세스하는 데에 걸리는 시간과 time stamp counter MSR에 액세스하는 데에 걸리는 시간은 다를 것입니다. 반면에 기능적으로 동등한 MSR은 각 MSR에 대한 microcode가 대략 동일하기 때문에 거의 동일한 액세스 시간을 가질 것입니다. 예를 들어, `MTRR_PHYSBASE0`에 액세스하는 것은 `MTRR_PHYSBASE1`에 액세스하는 것과 비슷한 시간이 걸릴 것으로 예상됩니다.

이 접근 방식을 사용하면 register 액세스 시간을 비교하여 "유사" 및 "비유사" MSR을 구별할 수 있습니다. 물론 기능적으로 다른 두 MSR이 동일한 액세스 시간을 가질 수 있으므로 "유사" register는 인접한 register에 한해 정의하였습니다. 

*Global configuration register*에는 기능적으로 동일하거나 유사한 버전이 있을 가능성은 거의 없을 것으로 추측하였습니다. 대신, 이 register는 우리가 가정한 속성에 따라 고유(유일)할 것으로 예상하였습니다. 따라서 연구에서는 functionally unique MSR에만 초점을 맞추기 위해 functional family로 묶이는 MSR들을 후보에서 제거해나갔습니다. 예시로, 아래 그림에서 145h와 207h, 26bh 부분에 위치한 MSR은 각각 functional family를 형성하며 묶이니 후보에서 제합니다.

![side-channel attack_zoom](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/side-channel attack_zoom.png){: width="600" }

이러한 방법을 통해 VIA C3 프로세서에서 구현된 1300개의 MSR 중 43개의 functionally unique MSR을 식별하였습니다. 이로써 분석해야 할 숫자가 합리적인 선으로 줄었습니다.

다음 단계로, 43개의 후보 MSR 중 어떤 것이 *global configuration register*인지 확인하고, 특허 문헌에 따라, *DEC*를 활성화할 수 있는 새로운 x86 instruction(*launch instruction*)을 활성화하는 MSR bit를 찾아야 합니다.

43개의 MSR과 MSR당 64 bit이니 확인해야 할 bit는 총 2752개입니다. 대부분의 경우 문서화되지 않은 MSR bit를 toggle 시, general protection exceptions, kernel panics, system instability, system reset, total processor lock 등이 발생했습니다. 이러한 눈에 띄는 side effect가 있을 때마다 해당 bit는 후보에서 제외하였습니다.

Hardware system reset tool을 사용하여 후보 MSR bit toggle을 자동화했으며 bit toggle로 오류가 발생할 때마다 대상 시스템을 자동으로 reset 하였습니다. 일주일 동안 수백 번의 자동 재부팅을 통해 2752 bit 중 눈에 띄는 side effect 없이 toggle 할 수 있는 bit들을 확인했습니다.

![system for automatically determining the MSR bits](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/system for automatically determining the MSR bits.png){: width="600" }

확인한 모든 stable MSR bit를 활성화한 상태로, 추가된 새로운 instruction을 찾기 위해 [*sandsifter*](https://github.com/xoreaxeaxeax/sandsifter)를 사용했습니다. 1억 개 이상의 instruction을 스캔한 결과, 프로세서에서 정확히 하나의 새로운 예기치 않은 instruction인 `0f3f`가 발견됐습니다. 어떠한 verdor의 프로세서 문서에서도 이 instruction에 대한 내용를 찾을 수 없었습니다. 이것은 아마도 VIA 특허에서 암시된 *launch instruction*일 것입니다. 약간의 시행착오를 거쳐 GDB로 instruction을 관찰한 결과, *launch instruction*이 사실상 `jmp %eax` instruction 이라는 것을 확인하였습니다. 즉, `eax` register에 있는 주소로 분기됩니다.

![sandsifter result](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/sandsifter result.png){: width="600" }

*Launch instruction*이 식별되고 나면 앞서 찾아낸 stable MSR bit 중 *launch instruction*을 활성화한 것이 어떤 것인지 빠르게 알아낼 수 있습니다. 각 stable MSR bit를 하나씩 활성화하며 `0f3f` 실행을 시도하니, MSR 1107h, bit 0이 C3 프로세서에서 *launch instruction*을 활성화한다는 사실이 금세 드러났습니다.

따라서 *global configuration register*는 MSR 1107h였고, MSR 1107h의 bit 0는 *god mode bit*라고 명명하였습니다.

## The x86 Bridge

*God mode bit*를 발견하였고, *launch instruction*도 알아냈으니 이제는 RISC core에서 명령을 실행하는 방법을 찾아야 합니다. US8880851을 보면 *launch instruction* 이후의 instruction을 fetch 할 때 별도의 RISC pipeline으로 보내지는 것으로 나타납니다.

![US8880851-fig2](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/US8880851-fig2.png){: width="600" }

그러나 대상 프로세서를 조사한 결과, 그렇지 않은 것으로 보였습니다. *God mode bit*를 활성화하고 *launch instruction*을 실행해도 프로세서가 계속 x86 instruction을 실행하는 것처럼 보였습니다.

상당한 시행착오를 거친 후, *launch instruction*이 decoder를 직접 전환하는 대신, x86 decoder가 첫 번째 decode pass를 수행하고 decoded instruction의 일부를 두 번째 RISC decoder로 전송하도록 x86 decoder 내의 기작을 변경할 수 있다는 가설을 세웠습니다. 이 구현에서 pipeline은 특허에 표시된 것처럼 instruction fetch phase 직후에 나뉘지 않고, 대신 x86 decoder 내에서 분기됩니다.

![dual execution pipeline](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/dual execution pipeline.png){: width="600" }

위 그림에 나타나는 것과 같이, instruction cache에서 x86 pre-decoder로 instruction이 전달된 후 pre-decoder는 instruction을 prefix, opcode, modr/m, scale-index-base, displacement, immediate bytes와 같은 구성 요소로 나눕니다. 그리고 이 시점에서 검사가 수행됩니다: 프로세서가 RISC mode(= 직전에 *launch instruction*이 실행된 경우)에 있고 instruction이 32-bit immediate value를 사용하며 나머지 구성 요소가 구조적으로 정의된 값과 일치하면 32-bit immediate은 RISC decoder로 전달됩니다.

이 같은 구현에서 RISC core에 32-bit immediate value를 전달하는 데에 사용되는 x86 instruction을 알아내야 합니다. 이 instruction은 두 개의 core를 연결하기 때문에 *bridge instruction*으로 명명하였습니다. 예시로, *bridge instruction*이 `mov eax,xxxxxxxx`이라고 할 때, `xxxxxxxx`는 RISC core가 활성화된 경우 RISC core로 전송되는 32-bit immediate value입니다.

RISC instruction의 형식을 모르기 때문에 *bridge instruction*을 알아내기 위해서는 x86 core에서의 간접적인 관찰을 통해 유추해야 합니다. RISC core가 실제로 권한 우회 메커니즘을 제공하는 경우 ring 3에서 실행되는 일부 RISC instruction은 시스템을 손상할 수 있어야 하며(invalid value를 control register 또는 kernel memoty에 쓰는 등) 이러한 시스템 손상은 processor lock, kernel panic, system reset의 형태로 감지될 수 있습니다. Unprivileged x86 instruction은 일반적으로 processor lock, kernel panic, system reset을 유발할 수 없으므로 unprivileged x86 instruction을 실행할 때 이러한 동작 중 하나가 관찰되면 *bridge instruction*이라고 판단할 수 있습니다.

이러한 접근법을 사용한다면 *sandsifter* tool을 적용하여 random processor fuzzing을 통해 *bridge instruction*을 찾을 수 있습니다: 1. *God mode bit* 설정. 2. *Launch instruction*에 이어 random x86 instruction 실행. 3. 반복.

1시간 이내로 fuzzing 한 결과, *bridge instruction*은 `bound %eax,0x00000000(,%eax,1)`으로 결정되었으며 여기서 `0x00000000`은 *DEC*로 전송되는 32-bit RISC instruction에 해당합니다. *Bridge instruction*은 microarchitecture에 따라 달라지며 `bound` bridge는 VIA C3 Nehemiah core에서 유효합니다.

## A Deeply Embedded Instruction Set

*어떻게* *DEC*에서 instruction을 실행하는지 알았으니, 다음으로는 *무엇을* 실행할지 알아야 합니다. 현재 상태로서는 *DEC*에서 실행되는 instruction이 어떻게 생겼는지, 어떤 architecture를 따르는지 따위도 알지 못합니다. 

처음에는 ARM, PowerPC, MIPS와 같은 RISC architecture의 간단한 instruction이 big 및 little endian 형식으로 시도됐습니다(예를 들어, ARM의 경우 `ADD R0,R0,#1`). 몇 번의 시도 후, instruction을 알려진 architecture와 명확하게 일치시키는 것은 어렵지만 architecture를 배제하는 것은 가능하다는 것을 깨달았습니다. *DEC*로 전송된 많은 instruction이 processor lock을 유발했습니다. `ADD R0,R0,#1`과 같이 후보 architecture에 대해 간단한 non-locking instruction을 실행한 후 processor lock이 유발된 경우 해당 architecture를 배제할 수 있습니다. 이를 통해 30개의 architecture가 배제됐고 이는 사실상 시도한 모든 알려진 architecture가 배제된 것이었습니다.

Core를 알려진 architecture와 일치시킬 수 없었기에 *deep embedded instruction set(DEIS)*으로 명명한, *DEC*에 대한 instruction set을 리버싱해야 했습니다.

이러한 instruction의 형식을 이해하려면 RISC instruction을 실행하고 결과를 관찰해야 합니다. 그러나 RISC instruction set에 대한 지식 없이는 RISC core를 직접적으로 관찰할 수 없습니다. 대신, US8880851에 따라서, x86 core와 RISC core가 부분적으로 공유된 register file을 가진다는 사실을 이용했습니다. 이를 통해 x86 core에서 RISC instruction의 결과 중 일부를 관찰할 수 있었으며 RISC instruction 형식을 해독할 수 있었습니다.

![US8880851-fig3](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/US8880851-fig3.png){: width="600" }

접근 방식은 다음과 같습니다. 우선, *god mode bit*를 설정합니다. 그 후 system *input state*를 생성하고 이를 기록합니다. System *input state*는 processor register state(general purpose register, special purpose register, MMX register)와 현재 userland process 및 kernel memory buffer로 구성됩니다. x86 *bridge instruction*으로 wrapping 된 임의의 RISC instruction을 *DEC*에서 실행할 수 있도록 *launch instruction*과 함께 실행합니다. GPR, SPR 및 MMX register와 userland 및 kernel memory buffer를 포함한 system *output state*를 기록합니다. 기록한 *input state*와 output state를 *diffing*하여 unknown RISC instruction에 대한 정보를 얻습니다.

![fuzzing DEIS](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/fuzzing DEIS.png){: width="600" }

다수의 node로 3주 동안 fuzzing 한 결과, 2301295개의 state diff로 구성된 15GB의 로그가 근 4000시간의 compute time에 걸쳐 축적되었습니다.

Test instruction은 처음에는 큰 baseline dataset을 얻기 위해 무작위로 생성되었습니다. Fuzzing의 initial round에서는 x86/RISC system state의 극히 일부만 기록할 수 있었기 때문에 대부분의 RISC instruction으로부터 별다른 결과를 얻지 못하였습니다. 이를 극복하기 위해 단계적 fuzzing 접근법이 사용되었습니다: System state에 가시적인 영향을 미치는 첫 번째 round의 instruction이 두 번째 fuzzing round에서 seed instruction으로 사용되었습니다. 이 단계적 접근법은 관찰 가능한 instruction 결과를 향상해 dataset의 완성도를 크게 높였습니다.

큰 corpus의 state diff를 사용하여 instruction 규칙을 식별하는 과정을 자동화하기 위해 *collector*라는 도구를 설계하였습니다. 이는 arithmetic operation과 memory access와 같은 다양한 일반적인 instruction effect에 대한 state diff를 확인합니다.

![collector1](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/collector1.png){: width="600" }

이후 여러 가지 heuristic을 통해 아래와 같은 instruction bin과 각 bin에 대한 binary encoding을 도출하였습니다. 이는 *collector*에 의해 식별된 instruction 카테고리의 일부일 뿐이지만, *DEC*에 대한 proof-of-concept privilege escalation attack을 수행하기에 충분하였습니다. 추가적인 분석을 통해 *collector*의 결과를 완전히 활용한다면 더 많은 *DEIS*를 재구성하여 *DEC*에서 범용 RISC computation을 가능하게 할 것으로 사료됩니다.

![collector2](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/collector2.png)

Register는 4 bit로 인코딩되며, `eax`는 `0b0000`, `ebx`는 `0b0011`, `ecx`는 `0b0001`, `edx`는 `0b0010`, `esi`는 `0b0110`, `edi`는 `0b0111`, `ebp`는 `0b0101`, `esp`는 `0b0100`입니다. Register 인코딩의 상위 bit는 RISC-only 또는 MMX register를 선택하는 데 사용되는 것으로 추측됩니다.

Instruction은 0, 1 또는 2개의 explicit register에서 작동합니다. 0 또는 1개의 explicit register에서 작동할 때 `eax` register는 때때로 implicit register로 사용됩니다. 0에서 8개의 opcode bit는 일반적으로 instruction의 시작 부분에 나타나며 instruction의 다른 위치에 추가 opcode bit가 있을 가능성이 있습니다.

## Privilege Escalation Payload

Proof-of-concept으로서, *rosenbridge* backdoor payload를 하나 제작하였습니다. 이 payload는 unprivileged userland process에서 실행되어 kernel memory를 읽고 수정하는 instruction을 *DEC*로 전달하고 root 권한을 획득합니다.

![poc-overview](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/poc_overview.png){: width="600" }

위 그림은 payload에 대한 개요를 나타냅니다. Kernel memory에서 x86 global descriptor table(GDT)을 읽어 현재 프로세스의 `task_struct` structure에 대한 pointer를 찾습니다. `task_struct`에서 프로세스의 `cred` structure에 대한 pointer를 찾고, `cred` structure에 접근해 root 권한 값을 설정합니다.

아래 코드는 privilege escalation payload에 대한 pseudocode입니다. 구현된 payload는 Debian 6.0.10 (i386), Linux kernel version 2.6.32를 기반으로 작성되었습니다.

![poc_pseudocode](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/poc_pseudocode.png){: width="600" }

Pseudocode를 상기 "A Deeply Embedded Instruction Set" 섹션에서 설명한 backdoor primitive로 변환하여야 합니다. 분석된 일부 primitive만을 가지고 원하는 행위를 구현하는 것은, ROP chain을 작성할 때와 같이 약간의 창의성을 요구합니다. Custom assembly language로 작성한 최종 payload는 다음과 같습니다.

![poc_assembly](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/poc_assembly.png){: width="600" }

마지막으로, prototype을 실제 작동하는 실행 파일로 변환합니다. `0f3f` *launch instruction*으로 *DEC*를 활성화하고 x86 'bound' *bridge instruction*을 통해 *DEIS* instruction을 실행하도록 합니다.

![poc](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/poc.png){: width="600" }

Pwned:)

![demo](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/demo.png){: width="600" }

# Conclusion

*rosenbridge* backdoor는 x86 프로세서에서 알려진 최초의 hardware level 백도어입니다. 이것은 그 자체로 보안 연구에서 급진적인 발전입니다. 그러나 백도어는 백서에서 살펴본 바와 같이 예전 프로세서 라인의 일부에만 나타납니다. 오늘날의 일반 사용자에게는 위협이 되지 않습니다. 대신, 본 연구의 주요 가치는 하드웨어 백도어의 가능성(결정적으로 입증된)에 대한 사례 연구, 그러한 백도어가 구현되는 방법에 대한 실질적인 조사 및 외부 관찰자가 위협을 발견하는 방법에 대한 사고 실험입니다.

하드웨어 백도어의 여파로 기존 보안 모델이 거의 완전히 망가졌습니다. 소프트웨어 보호 메커니즘에 대한 수십 년의 노력은 이러한 위협으로부터 보호하는 데 아무런 도움이 되지 않습니다. 이에 저자는, 이러한 사실에 당황하고 추측하기보다는 프로세서를 검사하고 감사하는 도구를 계속 개발하여 칩의 최종 사용자에게 제어력과 통찰력을 다시금 제공하는 것이 바람직한 방향일 것이라고 제안합니다.

*Christopher Domas의 Black Hat 발표 자료 중 발췌.

![conclusion](/assets/upload/2023-02-02-Ring-4-Hardware-Backdoors-in-CPUs/conclusion.png)

# Glossary

***bridge instruction***\
32-bit immediate value를 포함한 standard x86 instruction으로, *launch instruction*이 선행되면 32-bit immediate를 *deeply embedded core*의 RISC pipeline으로 보낸다. VIA C3 Nehemiah core에서 *bridge instruction*은 `bound %eax,xxxxxxxx(,%eax,1)`이며 여기서 `xxxxxxxx`는 RISC core로 전송되는 32-bit value이다.

***deeply embedded core (DEC)***\
프로세서의 x86 core와 함께 내장된 RISC core. RISC core는 x86 core와 긴밀히 통합되어 execution pipeline 및 register file의 중요한 부분을 공유한다.

***deeply embedded instruction set (DEIS)***\
*Deeply embedded core*에서 사용하는 instruction set.

***global configuration register***\
*God mode bit*를 포함하는 x86 model specific register.

***god mode bit***\
설정된 경우 *launch instruction*을 사용할 수 있는 x86 model specific register.

***launch instruction***\
*God mode bit*로 활성화된 새로운 x86 instruction. Instruction `0f3f`가 *deeply embedded core*를 활성화한다.

***rosenbridge***\
x86 프로세서의 백도어.

***sandsifter***\
x86 프로세서의 instruction set을 철저히 스캔하는 소프트웨어 도구로, 문서화되지 않은  instruction을 찾는 데 사용할 수 있다.  

<br>

*수정 및 보충 사항은 [dominic2009@snu.ac.kr](mailto:dominic2009@snu.ac.kr)로 제보 바랍니다.*

---

[^1]: [https://www.bloomberg.com/news/features/2018-10-04/the-big-hack-how-china-used-a-tiny-chip-to-infiltrate-america-s-top-companies](https://www.bloomberg.com/news/features/2018-10-04/the-big-hack-how-china-used-a-tiny-chip-to-infiltrate-america-s-top-companies)
[^2]: [https://www.blackhat.com/us-18/briefings/schedule/#god-mode-unlocked---hardware-backdoors-in-x86-cpus-10194](https://www.blackhat.com/us-18/briefings/schedule/#god-mode-unlocked---hardware-backdoors-in-x86-cpus-10194)
[^3]: 이후 [DEF CON 26](https://youtu.be/jmTwlEh8L7g), [HITB GSEC SINGAPORE 2018](https://gsec.hitb.org/sg2018/sessions/god-mode-unlocked-hardware-backdoors-in-x86-cpus/)에서도 발표함.
[^4]: [US8880851](https://patents.google.com/patent/US8880851), [US9292470](https://patents.google.com/patent/US9292470), [US9317301](https://patents.google.com/patent/US9317301)
[^5]: [US9043580](https://patents.google.com/patent/US9043580), [US9141389](https://patents.google.com/patent/US9141389), [US9146742](https://patents.google.com/patent/US9146742)
[^6]: [https://github.com/xoreaxeaxeax/rosenbridge/issues/17](https://github.com/xoreaxeaxeax/rosenbridge/issues/17)    
