# BlueZ LE Audio 수정 — acceptor(서버) 역할

BlueZ 5.87에 대한 패치 네 개입니다. BlueZ를 LE Audio 유니캐스트 링크의
**acceptor** 쪽으로 돌리다가 찾았습니다. 헤드셋 스택이 흔히 맡는 initiator가
아니라, 라우터가 휴대폰의 오디오 싱크·소스 노릇을 하는 구성입니다.

acceptor 경로는 상대적으로 덜 쓰이고, 아래는 그 경로가 버티지 못하는 지점들
입니다. 각 패치마다 어떤 증상에서 출발했는지, 왜 생기는지, 어떻게 재현하는지를
적었습니다.

이 패치들을 찾을 때 BlueZ 위에 올라가 있던 것은
[asterisk-chan-mobile-leaudio][chan]입니다. Asterisk `chan_mobile`용 LE Audio
유니캐스트 전송 계층인데, 증상이 처음 드러난 곳이 거기입니다 — 통화는 연결되고
오디오는 없고 에러도 없는 상태. 223 재현의 나머지 절반이기도 합니다.

[chan]: https://github.com/YDaBang/asterisk-chan-mobile-leaudio

*English version: [README.md](README.md)*

## 패치 목록

| 번호 | 위치 | 고치는 것 |
|------|------|-----------|
| 220 | `src/shared/bap.c` | 외부 PAC 소유자가 재시작해도 PAC context 재계산이 살아남음 |
| 221 | `profiles/audio/media.c` | 소유자 재시작에도 프로파일 서비스가 유지됨 |
| 222 | `src/shared/bap.c` | Assigned Generic Audio 메타데이터를 검증함 |
| 223 | `src/shared/bap.c` | 전송 IO가 붙어 있을 때 서버 쪽 Release가 완료됨 |

223을 먼저 보시면 됩니다. 멈춤(hang) 현상이고, 기존 단위 시험 틀에서 재현
되며, 해당 코드는 BAP 지원이 처음 들어온 이후 한 번도 바뀌지 않았습니다.

## 223 — 서버 쪽 Release가 끝나지 않음

### 증상

통화가 끝납니다. 상대가 스트림을 Release합니다. 그런데 ASE가 `RELEASING`을
벗어나지 못해서 다음 통화를 세울 수 없습니다. 안드로이드에서는 약 3.5초 뒤
watchdog이 그룹을 inactive로 내리고, 그 뒤로는 링크를 다시 세우기 전까지
휴대폰이 CIS를 제안하지 않습니다.

**오류 로그가 하나도 안 남습니다.** 양쪽 다 자기 기준으로는 정상 동작이라고
믿습니다.

### 원인

`stream_release()`는 스트림에 IO가 붙어 있지 않을 때만 상태 기계를 끝까지
몰고 갑니다.

```c
if (!stream->io) {
        ...
        stream_set_state(stream, BT_BAP_STREAM_STATE_RELEASING);
        if (cache_config)
                stream_set_state(stream, BT_BAP_STREAM_STATE_CONFIG);
}
```

IO가 붙어 있으면 완료를 `stream_io_disconnected()`에 맡깁니다. 이 콜백은
`bap_stream_io_attach()`가 등록하고, 전송 소켓이 끊길 때 불립니다.

**initiator라면 문제가 없습니다.** 자기가 Release를 시작했으니 전송도 스스로
내리고, 그래서 콜백이 옵니다.

**acceptor는 다릅니다.** Release를 원격이 요청합니다. 이쪽에서는 아무도 전송을
내리지 않으니 콜백이 영영 오지 않고, 스트림은 `RELEASING`에 갇힙니다.

`bap_stream_state_changed()`에도 같은 비대칭이 있습니다. RELEASING 처리에서
IO를 떼는 부분이 `if (stream->client)`로 막혀 있습니다.

### 수정

서버 경로에서도 전이를 완료합니다. IO를 떼고 `CONFIG`로 정착시킵니다.
클라이언트 동작은 그대로입니다.

### 재현 방법

패치가 `unit/test-bap.c`에 `BAP/USR/SCC/IO-RELEASE-01`을 추가합니다. source
스트림을 구성하고 `STREAMING`까지 몰고 간 뒤 `bt_bap_stream_set_io()`로 IO를
붙이고, 원격이 Release를 보내게 합니다.

**수정 없이는 이 시험이 멈춥니다.** 기대하는 Codec Configured 통지가 오지
않기 때문입니다.

등록은 `test_server`가 아니라 `test_server_state`로 해야 합니다. 앞의 것만
`cfg->state_func`를 설치합니다. (이걸 쓰다가 알게 된 건데, 기존 서버 쪽
구성들이 `state_func`를 지정해놓고 `test_server`로 등록돼 있어서 그 콜백들이
실제로는 실행되지 않고 있습니다.)

## 220, 221 — 외부 소유자 재시작 견디기

PAC 레코드와 미디어 엔드포인트를 `bluetoothd` 안에 넣지 않고 별도 프로세스가
D-Bus로 등록하는 구성에서 문제가 됩니다. 그 프로세스가 재시작하면 BlueZ는
상대가 아직 있다고 믿는 상태를 버리는데, 상대에게는 알리지 않아서 상대는 낡은
정보를 계속 들고 있게 됩니다.

221은 상류의 UAF 수정(`profiles/audio: fix UAF on external media service
teardown`)과 다른 문제입니다. 그쪽은 teardown 중 큐 일관성을 지키는 것이고,
이쪽은 소유자가 재시작해도 등록된 서비스를 살려두는 것입니다.

## 222 — 메타데이터 검증

Assigned Generic Audio 메타데이터 LTV가 Assigned Numbers 문서에 정의된
타입·길이 조합을 확인하지 않고 통과했습니다. 잘못된 메타데이터는 그대로
넘기지 말고 정의된 ASCS 응답 코드(`0x0A` Unsupported Metadata, `0x0C` Invalid
Metadata)로 답해야 합니다.

## 시험 결과

    BlueZ 5.87
    unit/test-bap: 네 개 모두 적용한 상태에서 709/709 통과
    패키지 빌드 재현성: 2회 실행 byte-identical

223은 실제 장비에서도 확인했습니다. MediaTek 컨트롤러가 Galaxy S23의 LE Audio
acceptor로 32 kHz LC3 유니캐스트 링크를 맡는 구성입니다. 수정 전에는 통화
때마다 ASE가 갇혀 다음 통화가 실패했고, 수정 후에는 연속 두 통화가 모두 깨끗
하게 해제되고 그룹이 활성 상태로 유지됐습니다.

## 적용 방법

이 패치들은 더 긴 시리즈의 뒷부분입니다. 220~223은 OpenWrt의 bluez 패키지가
201~219로 들고 있는 상류 수정들 위에 얹힙니다. 그중 상당수는 이미 mainline에
들어간 커밋의 백포트입니다(`3f283c8e0`, `5c1c679ec`, `13b14db95`,
`7f826d003` 등). 맨 5.87 트리에 바로 적용하면 220과 222는 자리가 맞지 않습니다.

`git apply`가 아니라 `patch(1)`을 쓰십시오. 앞에 무엇이 오느냐에 따라 offset이
180줄 가까이 밀리는데, `git apply`는 fuzz를 전혀 허용하지 않습니다.

    git clone git://git.kernel.org/pub/scm/bluetooth/bluez.git
    cd bluez && git checkout 5.87
    for p in /path/to/patches/*.patch; do patch -p1 < "$p"; done

깨끗한 5.87 체크아웃에서 확인했습니다. 22개 전체 시리즈가 실패 0으로 적용되고
223이 `src/shared/bap.c`에 반영됩니다.

223만 필요하면 맨 5.87 트리에 단독으로 적용됩니다.

## 상태

상류에 제출하지 않았고, 제출할 계획도 없습니다. 필요한 사람이 쓸 수 있도록
분석과 재현 방법을 남겨두려고 공개합니다.

이 중 무엇이든 `linux-bluetooth`에 보내고 싶으시면 그냥 가져가십시오. 출처
표기도 허락도 필요 없습니다. 특히 223은 그 자체로 완결되고 수정이 없으면
실패하는 시험을 들고 있어서, 보내기 어렵지 않을 것입니다.

## 라이선스

BlueZ를 수정하는 패치이므로 같은 조건인 GPL-2.0-or-later로 제공합니다.
