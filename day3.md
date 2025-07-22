앞에서 grub이 커널을 로드하는 과정을 알아봤고, grub을 가상 이미지에 설치해서 커널을 실제로 실행해 보았다.
오늘은 GRUB이 어떤 파라미터를 커널로 전달하는지 코드를 통해 알아보자.
일반적으로 PC는 16비트 리얼모드로 부팅이 되는데, 이 16bit 명령어를 통해 우리는 BIOS에 명령을 내리게 되고, 이 과정에서 하드웨어와 관련된 많은 정보를 얻게 된다. 
현재 시스템의 PCI 장치수, 메모리 크기, 화면 해상도 등. 
GRUB은 이 과정을 직접 수행하고 시스템을 보호 모드로 전환하기에, 실행된 커널 입장에서는 하드웨어 관련 정보가 없는 상태이다. 
따라서 GRUB이 넘겨주는 MultiBootInfo라는 구조체를 통해 관련 정보를 전달받는다.
아무튼 이 전체적인 과정을 통해서 GRUB을 통해 커널 엔트리를 호출할 수 있게 되었다. 
GRUB이 없었다면 부팅코드 제작을 위한 어셈블리 코드 작성, 그리고 디스크에서 커널을 읽어들이기 위한 루틴을 모두 구현했어야 한다.

커널 호출 준비는 마쳤지만, 커널 엔트리 포인트(커널 함수 시작점)을 호출하려면 커널이 특정 명세를 만족해야 한다. GRUB은 커널 파일의 최초 80KB 부분을 검색해 특정 시그니처를 찾아냄으로써 이를 확인한다.
이 시그니처를 '멀티부트 헤더 구조체'라 부른다. 
_declspec(naked) void multiboot_entry(void){} 형태로 진행된다. 이 함수의 첫 부분은 실행코드가 아니라 데이터이기 때문에 정상 루틴으로는 절대 실행되지 않는다. GRUB은 이 구조체 내부 MULTIBOOT_HEADER_MAGIC 시그니처를 찾아 자신의 정의값과 같은지 체크한다.
정의값은 0x1BADB002이다.
커널임을 확정지었다면 멀티부트 헤더값 중 엔트리 주소값을 담은 멤버값을 읽어와 해당 주소로 점프한다. 이는 kernel_entry: 레이블이 존재하는 부분이며 여기서부터 실제 커널코드가 실행된다.
이 어셈블리 코드에서 커널 스택을 초기화하고 멀티부트 정보를 담은 구조체 주소와 매직넘버를 stack에 담아서 실제 커널 엔트리인 kmain을 호출한다.
여기서부터는 C++ 플로우 차트를 그려보면 좋다. 지금은 간단한 형태이니 설명해 보겠다.
kmain.cpp 코드는 아래와 같다.
#include "kmain.h"

_declspec(naked) void multiboot_entry(void)
{
	__asm {
		align 4
  
		multiboot_header:
		//멀티부트 헤더 사이즈 : 0X20
		dd(MULTIBOOT_HEADER_MAGIC); magic number
		dd(MULTIBOOT_HEADER_FLAGS); flags
		dd(CHECKSUM); checksum
		dd(HEADER_ADRESS); //헤더 주소 KERNEL_LOAD_ADDRESS+ALIGN(0x100064)
		dd(KERNEL_LOAD_ADDRESS); //커널이 로드된 가상주소 공간
		dd(00); //사용되지 않음
		dd(00); //사용되지 않음
		dd(HEADER_ADRESS + 0x20); //커널 시작 주소 : 멀티부트 헤더 주소 + 0x20, kernel_entry
			
		kernel_entry :
		mov     esp, KERNEL_STACK; //스택 설정

		push    0; //플래그 레지스터 초기화
		popf

		//GRUB에 의해 담겨 있는 정보값을 스택에 푸쉬한다.
		push    ebx; //멀티부트 구조체 포인터
		push    eax; //매직 넘버

		//위의 두 파라메터와 함께 kmain 함수를 호출한다.
		call    kmain; //C++ 메인 함수 호출

		//루프를 돈다. kmain이 리턴되지 않으면 아래 코드는 수행되지 않는다.
		halt:
		jmp halt;
	}
}

void InitializeConstructor()
{
	//내부 구현은 나중에 추가한다.
}

void kmain(unsigned long magic, unsigned long addr)
{
	InitializeConstructor(); //글로벌 객체 초기화

	SkyConsole::Initialize(); //화면에 문자열을 찍기 위해 초기화한다.

	SkyConsole::Print("Hello World!!\n");

	for (;;); //메인함수의 진행을 막음, 루프
}
