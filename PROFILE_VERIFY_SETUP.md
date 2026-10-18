# NIKKE Gear Manager - BlaBlaLink 세션 갱신 및 유효성 확인

본섭과 테스트 사이트는 Cloudflare Worker `nikke-profile-verify`를 사용해 공개 BlaBlaLink 프로필의 NIKKE 데이터를 조회합니다. Worker에는 자동 세션 갱신, 갱신된 쿠키를 KV에 저장하는 기능, 읽기 전용 세션 유효성 검사 기능이 포함되어 있습니다.

## 제공 기능

- Cloudflare Cron Trigger가 매일 한국 시간 12:00에 서비스 세션 갱신을 시도합니다.
- 계정 동기화를 실행할 때도 세션 갱신을 시도합니다.
- 사이트의 **서비스 세션 유효성 확인 · 계정 데이터 변경 없음** 버튼은 세션 갱신 후 실제 NIKKE 데이터 API 인증을 읽기 전용으로 확인합니다.
- 유효성 확인 버튼은 사이트의 계정 데이터나 장비 현황을 변경하지 않습니다.
- 만료된 세션이 완전히 폐기됐거나 BlaBlaLink가 재로그인을 요구하면 자동으로 복구되지 않을 수 있습니다.

## Cloudflare Worker 설정

Worker 이름은 `nikke-profile-verify`입니다. `wrangler.toml`은 다음 KV 네임스페이스와 예약 실행 설정을 사용합니다.

    [[kv_namespaces]]
    binding = "SERVICE_SESSION_KV"
    id = "288644a46eb343aa9928ec7a70e342c0"

    [triggers]
    crons = ["0 3 * * *"]

위 KV ID는 이 프로젝트에서 만든 네임스페이스입니다. 이미 해당 ID를 사용하고 있으므로 중복으로 만들지 마세요.

## 필요한 Secret

서비스용 BlaBlaLink 계정의 동일한 로그인 세션에서 아래 쿠키 값을 가져와 Worker Secret으로 저장합니다. 비밀번호나 쿠키 값을 GitHub 저장소나 채팅에 올리지 마세요.

| Worker Secret | 브라우저 쿠키 |
| --- | --- |
| `BLABLALINK_GAME_TOKEN` | `game_token` |
| `BLABLALINK_GAME_OPENID` | `game_openid` |
| `BLABLALINK_GAME_UID` | `game_uid` |
| `BLABLALINK_GAME_ADULT_STATUS` | `game_adult_status` |
| `BLABLALINK_GAME_CHANNELID` | `game_channelid` |
| `BLABLALINK_GAME_GAMEID` | `game_gameid` |
| `BLABLALINK_GAME_USER_NAME` | `game_user_name` (선택 사항; 쿠키가 비어 있으면 생략) |

Secret을 변경하려면 PowerShell에서 아래와 같이 실행합니다. 각 명령 뒤에 Wrangler가 값을 요청하면 해당 쿠키의 Value만 입력하세요.

    npx.cmd wrangler secret put BLABLALINK_GAME_TOKEN
    npx.cmd wrangler secret put BLABLALINK_GAME_OPENID
    npx.cmd wrangler secret put BLABLALINK_GAME_UID
    npx.cmd wrangler secret put BLABLALINK_GAME_ADULT_STATUS
    npx.cmd wrangler secret put BLABLALINK_GAME_CHANNELID
    npx.cmd wrangler secret put BLABLALINK_GAME_GAMEID

`BLABLALINK_GAME_USER_NAME` 쿠키가 비어 있다면 해당 Secret을 등록하지 않아도 됩니다. 기존 Secret이 없다는 이유로 `secret delete`를 실행할 필요도 없습니다.

## Worker 배포

`cloudflare-worker.js`와 `wrangler.toml`을 로컬 Worker 배포 폴더에 반영한 뒤 다음 명령을 실행합니다.

    npx.cmd wrangler deploy

배포가 성공하면 다음 명령으로 갱신 결과를 확인할 수 있습니다.

    npx.cmd wrangler tail nikke-profile-verify

성공 로그는 `[service-session] on-demand refresh succeeded`, 예약 갱신 성공 로그는 `[service-session] scheduled refresh succeeded`입니다. 세션 사전 검사에서는 `[service-session] preflight API validation succeeded`를 확인할 수 있습니다.

## 사이트에서 사전 검사

1. 사이트를 강력 새로고침합니다(`Ctrl + F5`).
2. **계정 동기화** 창을 엽니다.
3. 공개 BlaBlaLink 프로필 URL을 입력합니다.
4. **서비스 세션 유효성 확인 · 계정 데이터 변경 없음**을 누릅니다.

검사는 세션이 현재 유효한지 확인하는 용도이며, 이후 미래 시점의 만료를 완전히 보장하거나 예측하지는 않습니다.

## 보안

- 세션 값은 Cloudflare Secret과 Worker KV에만 저장합니다.
- 쿠키와 토큰을 소스 코드나 GitHub에 커밋하지 않습니다.
- 갱신 API가 세션을 거부하면 BlaBlaLink에 다시 로그인하고 최신 쿠키 값을 Secret으로 갱신해야 할 수 있습니다.
