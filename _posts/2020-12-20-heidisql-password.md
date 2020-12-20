---
title: HeidiSQL 비밀번호 찾기
categories:
    - miscellaneous
tags:
    - heidisql
    - decryption
    - python

---

### HeidiSQL 비밀번호 찾기

HeidiSQL은 비밀번호를 간단하게 암호화하여 레지스트리에 보관하고 있다.

HeidiSQL에 세션 정보를 저장해놓고 쓰다가 필요할 때 비밀번호가 기억 나지 않는다면 간단하게 코드를 짜서 복호화 할 수 있다.

1. HeidiSQL에서 세션을 아무거나 연다.
2. '파일' -> '설정 파일 내보내기...'를 눌러 설정 파일을 내보낸다. (모든 세션의 정보가 다 Export 되기 때문에 1번에서 아무 세션이나 열어도 상관 없다.)
3. Export 한 설정 txt 파일을 열어 'Password'단어로 검색하여 복호를 원하는 세션의 'Password' 라인을 찾느다. 맨 오른쪽 값이 암호화 된 비밀번호이다.
4. python에서 다음의 코딩을 한다.
    ```python
    def decode(hex):
        shift = int(hx[-1])
        l = [int(hx[i:i+2], 16) for i in range(0, len(hx), 2)]
        return ''.join(chr(v - shift) for v in l)
    ```

5. `print(decode([암호화 된 비밀번호]))` 를 하면 비밀번호가 출력된다.
    ```python
    print(decode('323334351'))
    ```

---
**NOTE**

1. 설정 내보내기를 하지 않고 레지스트리에서 저장된 비밀번호를 가져오고 싶다면 \
regedit을 연 뒤 HKEY_CURRENT_USER\Software\HeidiSQL\Servers 에서 원하는 세션을 찾아서 암호화 된 Password 값을 확인할 수 있다.
2. Portable 버전을 사용하는 경우에는 실행 파일이 있는 곳의 'portable_settings.txt'에 설정 파일이 저장되어 있으므로 \
위의 1~2번 과정을 할 필요 없이 바로 이 파일을 열어 봐도 된다.
---

> Cr. https://gist.github.com/jpatters/4553139#gistcomment-1880752
