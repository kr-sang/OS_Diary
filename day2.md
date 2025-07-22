2일차. 용어 정리부터 하고 시작.
skyos.ima 는 가상 디스크 이미지 파일이다. 이 디스크 이미지 내부에 skyos.exe나, grub 등이 들어 있다.
부팅 시도 시 grub등의 부트로더 디스크 이미지 내에 있는 exe 파일을 가상 플로피 디스크에 올린다.

부팅을 시도할 경우 우선 BIOS는 부팅 장치를 찾고, 그 장치에서 MBR을 읽어온다. 우리의 프로젝트를 예시로 보면, BIOS는 QEMU라는 가상 환경에서 부팅 장치인 skyos.ima(가상 디스크 이미지 파일)를 찾고, 해당 가상 디스크의 첫 섹터인 MBR에는 GRUB stage1 코드가 들어있게 된다. 이 코드로 인해 stage가 진행되며 stage2의 메뉴를 통해 사용자 지정 커널이 메모리에 적재되며 커널 엔트리가 실행된다.

좀 더 자세히 알아보자. 물리적 하드디스크는 여러 논리 하드 디스크로 나눌 수 있다. 그리고 MBR에는 최대 4개의 파티션 정보가 저장되며, 이들 중 하나의 파티션만이 부팅 가능(active) 플래그를 true로 가진다. 이렇게 찾은 부팅 가능한 파티션 내부에는 OS를 실행할 준비가 된 '부트 섹터'가 존재해야 한다.
BIOS는 부팅 가능한 파티션에서 부트 섹터를 로딩한다. 로딩된 부트 섹터의 코드는 커널 로더를 메모리로 로딩하게 된다. 로딩된 커널 로더가 커널 이미지인 skyos32.EXE 등을 실행시키며 초기화에 필요한 드라이버, 파일 시스템, 설정 파일 등을 불러와 RAM에 적재하고 OS의 실행을 준비한다.

계속 조금씩 언급하고 있지만, 16bit 리얼모드 처리 과정부터 커널 로딩 과정까지를 간편하게 도와주는 GRUB이라는 모듈이 있다. 우리는 이를 활용하여 커널을 메모리에 적재하는 과정을 처리한다.
winImage를 통해 새로운 가상 디스크 이미지를 만들어 주고, BootIce를 통해 해당 디스크 이미지의 PBR을 process한다. 이 과정에서 GRUB의 부트섹터 코드가 플로피 디스크의 부트섹터에 카피된다.
이후 이미지 안에 여러 파일들을 복사한다.
기존 sln 프로젝트에서 빌드한 skyos32.exe 파일과 .map 파일을 넣어주고, menu.lst, stage 1,2 파일이 담긴 boot 폴더와 grldr 파일을 넣어준다. 
이미지 파일이 준비 완료되었으므로 배치 파일을 만들어 준다.
코드는 아래와 같다.

REM Start qemu on windows.
@ECHO OFF
SET SDL_VIDEODRIVER=windib
SET SDL_AUDIODRIVER=dsound
SET QEMU_AUDIO_DRV=dsound
SET QEMU_AUDIO_LOG_TO_MONITOR=0
qemu-system-x86_64.exe -L . -m 128 -fda skyOS3.ima -soundhw sb16,es1370 -localtime -M pc

이후 cmd에서 실행하여 올바른 결과를 얻어냈다.

결국 오늘 한 일을 정리하면,
GRUB이 커널을 로드하는 과정을 알아보았고 가상 이미지에 GRUB을 설치하여 커널을 실제로 실행해 보았다.
