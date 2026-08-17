# 223 패치를 linux-bluetooth 에 보내는 법

## 보낼 것

    submission/0001-shared-bap-settle-server-side-Release-when-a-transpor.patch

깨끗한 BlueZ 5.87 체크아웃에 단독으로 적용되는 것을 확인했습니다.
`Signed-off-by` 가 `---` 위에 있어야 커밋 메시지에 남는데, 그 위치도 맞췄습니다.

## 받는 곳

    To:  linux-bluetooth@vger.kernel.org

메인테이너를 Cc 에 넣을 필요는 없습니다. 리스트를 다 봅니다.

## 중요 — 메일 형식

리눅스 계열 리스트는 **plain text 만** 받습니다. HTML 메일은 서버가 조용히
버립니다. 프로톤 웹에서 보내실 거면 작성 창 하단 `⋮` 메뉴에서 HTML 을 끄셔야
합니다.

그리고 **패치를 첨부파일로 보내지 마세요.** 본문에 그대로 붙여야 리뷰어가
인용하며 답할 수 있습니다.

## 방법 A — 프로톤 웹에서 직접 (간단함)

1. 위 패치 파일을 텍스트 편집기로 엽니다.
2. `Subject:` 줄의 내용을 메일 제목에 넣습니다.

       [PATCH BlueZ] shared/bap: settle server-side Release when a transport IO is attached

3. `From:` 과 `Subject:` 두 줄을 **뺀 나머지 전부**를 메일 본문에 붙입니다.
   `---` 와 diff 까지 전부 포함합니다.
4. plain text 모드인지 확인하고 보냅니다.

붙여넣을 때 편집기가 **줄바꿈을 바꾸거나 탭을 공백으로 만들면 패치가
깨집니다.** 보내기 전에 diff 부분의 들여쓰기가 원본과 같은지 눈으로 확인하세요.

## 방법 B — git send-email (형식이 안 깨짐)

프로톤은 일반 SMTP 를 열어주지 않아서 **Proton Mail Bridge** (유료 플랜) 가
필요합니다. 브리지를 쓰신다면:

    git config --global sendemail.smtpServer 127.0.0.1
    git config --global sendemail.smtpUser y_dabang@protonmail.com
    git config --global sendemail.smtpEncryption tls
    git config --global sendemail.smtpServerPort 1025

    git send-email --to=linux-bluetooth@vger.kernel.org \
        submission/0001-*.patch

브리지가 없으면 방법 A 로 하시면 됩니다. 형식만 지키면 결과는 같습니다.

## 보낸 뒤

메일이 몇 분 안에 아카이브에 뜹니다. 검색해서 형식이 깨지지 않았는지
확인하세요.

    https://lore.kernel.org/linux-bluetooth/

회신이 오면 대개 이런 것들을 물어봅니다.

- 재현 절차 (README 에 있는 IO-RELEASE-01 을 안내하면 됩니다)
- 클라이언트 동작에 영향이 없는지 (없습니다. `!stream->client` 로 막혀 있습니다)
- 왜 `stream_io_disconnected()` 를 고치지 않았는지
  (그 콜백은 소켓이 실제로 닫혀야 실행되는데, 서버 쪽은 아무도 닫지 않습니다)

답장이 필요하면 내용을 알려주세요. 영어로 써드리겠습니다.

## 답이 없으면

리스트는 조용할 때가 있습니다. **2주 정도 기다린 뒤 같은 스레드에 한 번
재전송(ping)** 하는 것이 관례입니다. 재촉하는 인상 없이 다시 눈에 띄게 하는
방법입니다.
