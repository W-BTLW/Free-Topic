# Github Actions를 활용한 숙제 검사하기

## 고민
- 돈워리 친구들이 숙제를 잘 하고 있는지 궁금했습니다.
- 숙제를 제출하면 **실시간으로** 그 결과를 알고 싶었습니다.
- 동기부여가 필요했습니다.

## 요구사항
- 서버 없이 최소한의 비용으로 기능 구현만 하고 싶음.
- 스터디원들은 모두 카카오톡을 사용하고 있음. 디스코드 ❌ 슬랙 ❌.
- 모두에게 알림이 가면 좋겠지만, 나만 봐도 상관 없음. 궁금한건 나니까.
- 커밋을 올렸다는 것은 숙제를 했다는 말임.

## 설계
- [x] GitHub Actions를 사용하면 서버를 따로 띄우지 않고도, 깃허브 서버에서 숙제(커밋)가 올라올 때마다 알림을 보내는 코드를 무료로 실행할 수 있었음.
- [x] GitHub Push ➡️ GitHub Actions 발동 ➡️ 카카오톡 "나에게 보내기" API 호출
- [ ] 디스코드나 슬랙은 카카오와는 다르게 연동이 잘 되어있는지 웹훅 주소만 레포지토리 설정에 넣으면 끝남 !

## 방법
<details> 
<summary> 카카오 개발자 설정(처음 1번만)</summary>
<br>

1. 카카오 개발자 센터 로그인 후 내 애플리케이션 > 애플리케이션 추가하기.

2. 제품 설정 > 카카오 로그인: '활성화 설정'을 ON으로 변경.

3. 카카오 로그인 > 동의항목: '카카오톡 메시지 전송(나에게 보내기)'을 필수 동의로 설정.

4. 앱 키 > **REST API 키**를 따로 메모해두세요.

<br>
</details>

<details> 
<summary> 초기 토큰 발급 (수동 1회)</summary>
<br>

1. 카카오 API는 처음에 사용자 로그인을 통해 '인증 코드'를 받아야 합니다.

2. 브라우저 주소창에 아래 주소를 입력하여 로그인하고 동의합니다.
```html
https://kauth.kakao.com/oauth/authorize?client_id={REST_API_키}&redirect_uri=https://localhost:3000&response_type=code&scope=talk_message
```

3. 로그인 후 주소창이 https://localhost:3000/?code=이부분이코드입니다 로 바뀝니다. 해당 코드를 복사하세요.

4. 터미널(CMD)에서 아래 명령어를 입력해 토큰을 받아옵니다.
```bash
curl -v -X POST "https://kauth.kakao.com/oauth/token" \
 -d "grant_type=authorization_code" \
 -d "client_id={REST_API_키}" \
 -d "redirect_uri=https://localhost:3000" \
 -d "code={방금_복사한_코드}"
```
5. 결과값 중 **refresh_token**을 반드시 안전한 곳에 저장해두세요. (이걸로 평생 자동 갱신이 가능합니다.)

<br>
</details>


<details> 
<summary> GitHub Secrets 설정</summary>
<br>

알림을 보낼 깃허브 레포지토리의 Settings > Secrets and variables > Actions 메뉴에서 다음 두 값을 등록합니다.

KAKAO_CLIENT_ID: 여러분의 **REST API 키**

KAKAO_REFRESH_TOKEN: 위에서 발급받은 **리프레시 토큰**

<br>
</details>

<details> 
<summary> GitHub Actions 파일 작성</summary>
<br>

레포지토리의 .github/workflows/kakao-notify.yml 파일을 만들고 아래 코드를 붙여넣으세요. 이 코드는 1. 토큰 갱신 > 2. 메시지 전송을 한 번에 처리합니다.

```yaml
name: KakaoTalk Homework Bot
on: [push]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Refresh Kakao Token and Send Message
        run: |
          # 1. 리프레시 토큰을 사용하여 새로운 액세스 토큰 발급
          RESPONSE=$(curl -s -X POST "https://kauth.kakao.com/oauth/token" \
            -d "grant_type=refresh_token" \
            -d "client_id=${{ secrets.KAKAO_CLIENT_ID }}" \
            -d "refresh_token=${{ secrets.KAKAO_REFRESH_TOKEN }}")
          
          ACCESS_TOKEN=$(echo $RESPONSE | jq -r .access_token)

          # 2. 나에게 메시지 보내기 실행
          curl -v -X POST "https://kapi.kakao.com/v2/api/talk/memo/default/send" \
            -H "Authorization: Bearer $ACCESS_TOKEN" \
            -d "template_object={
                  \"object_type\": \"text\",
                  \"text\": \"🔔 숙제 알림\n${{ github.actor }}님이 [${{ github.event.repository.name }}]에 커밋했습니다!\n내용: ${{ github.event.head_commit.message }}\",
                  \"link\": {
                    \"web_url\": \"https://github.com/${{ github.repository }}\"
                  },
                  \"button_title\": \"커밋 확인하기\"
                }"
```

<br>
</details>

## 결과
![githubactionkakaosample](https://github.com/user-attachments/assets/df1ee540-75d7-4f97-ae14-aae36b4699e9)

## 총평
디스코드나 슬랙이 카카오에 비해 생태계가 큰 이유를 알 수 있다. 카카오는 알림톡으로 push 기능을 이용하려면 유료 비즈니스 플랜을 써야하기 때문이다.
*2026.03.23*
