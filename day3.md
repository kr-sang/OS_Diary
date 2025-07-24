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
위에서 말한대로 multiboot_entry함수가 어셈블리어로 작성되어 있고(이 부분은 order.txt에서 0x100400에 위치하도록 조정) 그 아래에 기능 함수와 kmain함수가 위치한다.

kermel load address : 0x100000
align : 0x400
-> PE를 기술하기 위한 헤더가 선두에 있어야 해서 400 존재. 그럼에도 80KB(제약조건) 안에 들어가도록 0x400 즉 1024B(1KB). 
이러한 조정을 위해 order.txt파일에서 ?multiboot_entry@@YAXXZ 를 추가하여 vs 컴파일러가 이 함수를 선두에 배치시키도록 한다.
header address : 0x100400 -> multiboot entry

멀티부트 엔트리 함수를 찾았으니 grub이 헤더 크기만큼 건너뛰고 kernel_entry 레이블에서부터 코드를 실행. 

커널 엔트리 어셈블리 코드... 이 모든 코드들은 어디에 존재하는 거지? 그냥 디스크가 골라지면 그 디스크의 극초반에 위치한 정보들인건가?

kmain에서는 글로벌 객체를 초기화하는 함수를 실행하고(이는 뒷 내용에서 학습한다. 지금은 GRUB의 제약조건에 맞추어 80KB 이내에서 시그니처를 찾아야 하는데 글로벌 변수 사용 시 주소가 밀리기 때문.), SkyConsole이라는 개체에서 정의되는 함수인 initialize와 print를 실행하여 콘솔창에 Hello world를 띄운다.

	#include "kmain.h"

	_declspec(naked) void multiboot_entry(void)
	{
		__asm {
			align 4
			//구조체 멤버들에 대한 정의 파트.
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

다음으로 SkyConsole.cpp를 보자. 현 시점에서는 콘솔 로거를 싱글턴 객체로 만들 수 없으므로 비슷한 느낌이 나도록 namespace를 사용한다.


	#include "SkyConsole.h"
	#include <stdarg.h>
	#include <stdint.h>
	#include <string.h>
	#include "sprintf.h"


	extern "C" int _outp(unsigned short, int);

	void OutPortByte(ushort port, uchar value)
	{
		_outp(port, value);
	}

	namespace SkyConsole
	{
		static ConsoleColor m_Color;
		static ConsoleColor m_Text;
		static ConsoleColor m_backGroundColor;

		static uint m_xPos;
		static uint m_yPos;

		static ushort* m_pVideoMemory; //Just a pointer to video memory
		static unsigned int m_ScreenHeight;
		static unsigned int m_ScreenWidth;
		static unsigned short m_VideoCardType;

		void Initialize()
		{
			char c = (*(unsigned short*)0x410 & 0x30);//Detects video card type, vga or mono

			if (c == 0x30) //c can be 0x00 or 0x20 for colour, 0x30 for mono
			{
				m_pVideoMemory = (unsigned short*)0xb0000;
				m_VideoCardType = VGA_MONO_CRT_ADDRESS;	// mono
			}
			else
			{
				m_pVideoMemory = (unsigned short*)0xb8000;
				m_VideoCardType = VGA_COLOR_CRT_ADDRESS;	// color
			}

			m_ScreenHeight = 25;
			m_ScreenWidth = 80;

			m_xPos = 0;
			m_yPos = 0;

			m_Text = White;
			m_backGroundColor = Black;
			m_Color = (ConsoleColor)((m_backGroundColor << 4) | m_Text);

			Clear();
		}

		void Clear()
		{

			for (uint i = 0; i < m_ScreenWidth * m_ScreenHeight; i++)				//Remember, 25 rows and 80 columns
			{
				m_pVideoMemory[i] = (ushort)(0x20 | (m_Color << 8));
			}

			MoveCursor(0, 0);
		}

		void Write(const char *szString)
		{
			while ((*szString) != 0)
			{
				WriteChar(*(szString++), m_Text, m_backGroundColor);
			}
		}

		void WriteChar(char c)
		{
			WriteChar(c, m_Text, m_backGroundColor);
		}

		void WriteChar(char c, ConsoleColor textColour, ConsoleColor backColour)
		{
			int t;
			switch (c)
			{
			case '\r':                         //-> carriage return
				m_xPos = 0;
				break;

			case '\n':                         // -> newline (with implicit cr)
				m_xPos = 0;
				m_yPos++;
				break;

			case 8:                            // -> backspace
				t = m_xPos + m_yPos * m_ScreenWidth;    // get linear address
				if (t > 0) t--;
				// if not in home position, step back
				if (m_xPos > 0)
				{
					m_xPos--;
				}
				else if (m_yPos > 0)
				{
					m_yPos--;
					m_xPos = m_ScreenWidth - 1;
				}

				*(m_pVideoMemory + t) = ' ' | ((unsigned char)m_Color << 8); // put space under the cursor
				break;

			default:						// -> all other characters
				if (c < ' ') break;				// ignore non printable ascii chars
												//See the article for an explanation of this. Don't forget to add support for new lines

				ushort* VideoMemory = m_pVideoMemory + m_ScreenWidth * m_yPos + m_xPos;
				uchar attribute = (uchar)((backColour << 4) | (textColour & 0xF));

				*VideoMemory = (c | (ushort)(attribute << 8));
				m_xPos++;
				break;
			}

			if (m_xPos >= m_ScreenWidth)
				m_yPos++;

			if (m_yPos == m_ScreenHeight)			// the cursor moved off of the screen?
			{
				scrollup();					// scroll the screen up
				m_yPos--;						// and move the cursor back
			}
			// and finally, set the cursor

			MoveCursor(m_xPos + 1, m_yPos);
		}

		void WriteString(const char* szString, ConsoleColor textColour, ConsoleColor backColour)
		{
			if (szString == 0)
				return;

			ushort* VideoMemory = 0;

			for (int i = 0; szString[i] != 0; i++)
			{
				VideoMemory = m_pVideoMemory + m_ScreenWidth * m_yPos + i;
				uchar attribute = (uchar)((backColour << 4) | (textColour & 0xF));

				*VideoMemory = (szString[i] | (ushort)(attribute << 8));
			}

			m_yPos++;
			MoveCursor(1, m_yPos);
		}

		void Print(const char* str, ...)
		{

			if (!str)
				return;

			va_list		args;
			va_start(args, str);
			size_t i;

			for (i = 0; i < strlen(str); i++) {

				switch (str[i]) {

				case '%':

					switch (str[i + 1]) {

						/*** characters ***/
					case 'c': {
						char c = va_arg(args, char);
						WriteChar(c, m_Text, m_backGroundColor);
						i++;		// go to next character
						break;
					}

							  /*** address of ***/
					case 's': {
						const char * c = (const char *&)va_arg(args, char);
						char str[256];
						strcpy(str, c);
						Write(str);
						i++;		// go to next character
						break;
					}

							  /*** integers ***/
					case 'd':
					case 'i': {
						int c = va_arg(args, int);
						char str[32] = { 0 };
						itoa_s(c, 10, str);
						Write(str);
						i++;		// go to next character
						break;
					}

							  /*** display in hex ***/
							  /*int*/
					case 'X': {
						int c = va_arg(args, int);
						char str[32] = { 0 };
						itoa_s(c, 16, str);
						Write(str);
						i++;		// go to next character
						break;
					}
							  /*unsigned int*/
					case 'x': {
						unsigned int c = va_arg(args, unsigned int);
						char str[32] = { 0 };
						itoa_s(c, 16, str);
						Write(str);
						i++;		// go to next character
						break;
					}

					default:
						va_end(args);
						return;
					}

					break;

				default:
					WriteChar(str[i], m_Text, m_backGroundColor);
					break;
				}

			}

			va_end(args);
		}
		void GetCursorPos(uint& x, uint& y) { x = m_xPos; y = m_yPos; }

		void MoveCursor(unsigned int  X, unsigned int  Y)
		{
			if (X > m_ScreenWidth)
				X = 0;
			unsigned short Offset = (unsigned short)((Y*m_ScreenWidth) + (X - 1));

			OutPortByte(m_VideoCardType, VGA_CRT_CURSOR_H_LOCATION);
			OutPortByte(m_VideoCardType + 1, Offset >> 8);
			OutPortByte(m_VideoCardType, VGA_CRT_CURSOR_L_LOCATION);
			OutPortByte(m_VideoCardType + 1, (Offset << 8) >> 8);

			if (X > 0)
				m_xPos = X - 1;
			else
				m_xPos = 0;

			m_yPos = Y;
		}
		/* Sets the Cursor Type
			0 to 15 is possible value to pass
			Returns - none.
			Example : Normal Cursor - (13,14)
				  Solid Cursor - (0,15)
				  No Cursor - (25,25) - beyond the cursor limit so it is invisible
		*/
		void SetCursorType(unsigned char  Bottom, unsigned char  Top)
		{
		
		}

		void scrollup()		// scroll the screen up one line
		{
			unsigned int t = 0;

			//	disable();	//this memory operation should not be interrupted,
							//can cause errors (more of an annoyance than anything else)
			for (t = 0; t < m_ScreenWidth * (m_ScreenHeight - 1); t++)		// scroll every line up
				*(m_pVideoMemory + t) = *(m_pVideoMemory + t + m_ScreenWidth);
			for (; t < m_ScreenWidth * m_ScreenHeight; t++)				//clear the bottom line
				*(m_pVideoMemory + t) = ' ' | ((unsigned char)m_Color << 8);

			//enable();
		}

		void SetTextColor(ConsoleColor col)
		{						//Sets the colour of the text being displayed
			m_Text = col;
			m_Color = (ConsoleColor)((m_backGroundColor << 4) | m_Text);
		}

		void SetBackColor(ConsoleColor col)
		{						//Sets the colour of the background being displayed
			if (col > LightGray)
			{
				col = Black;
			}
			m_backGroundColor = col;
			m_Color = (ConsoleColor)((m_backGroundColor << 4) | m_Text);
		}

		unsigned char GetTextColor()
		{						//Sets the colour of the text currently set
			return (unsigned char)m_Text;
		}

		unsigned char GetBackColor()
		{						//returns the colour of the background currently set
			return (unsigned char)m_backGroundColor;
		}

		void SetColor(ConsoleColor Text, ConsoleColor Back, bool blink)
		{						//Sets the colour of the text being displayed
			SetTextColor(Text);
			SetBackColor(Back);
			if (blink)
			{
				m_Color = (ConsoleColor)((m_backGroundColor << 4) | m_Text | 128);
			}
		}
	}

커널 개발 시 제약사항은 타 파일에 작성하였음.
