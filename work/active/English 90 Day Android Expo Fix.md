---
date: "2026-07-07"
description: "english-90day-android Expo 앱에서 expo-asset 누락으로 Metro가 기동하지 않던 문제를 수정하고 검증한 작업 기록"
project: english-90day-android
status: active
quarter: Q3-2026
tags:
  - work-note
  - expo
  - android
  - react-native
source_path: /home/tim/work/english-90day-android
---

# English 90 Day Android Expo Fix

## Context

`/home/tim/work/english-90day-android`에서 `npm start` 실행 시 Expo Metro가 시작되지 않았다.

오류 메시지:

```text
Error: The required package `expo-asset` cannot be found
```

상황:

- Expo가 8081 포트 사용 중을 감지해서 8082 사용 여부를 물음
- 8082 사용을 선택한 뒤 Metro config 로딩 중 `expo-asset` resolve 실패
- stack trace는 `@expo/metro-config/src/ExpoMetroConfig.ts`의 asset plugin 로딩 경로를 가리킴

## Root Cause

`expo-asset`은 `expo` 패키지 내부 nested dependency로는 존재했지만, 프로젝트 top-level dependency에는 없었다.

확인 결과:

```text
node_modules/expo/node_modules/expo-asset/package.json
```

은 존재했지만, 다음 resolve는 실패했다.

```bash
node -e "require.resolve('expo-asset/tools/hashAssetFiles')"
```

Metro config가 프로젝트 기준으로 `expo-asset/tools/hashAssetFiles`를 resolve하려 하므로, 앱의 `package.json`에 `expo-asset`이 직접 dependency로 있어야 한다.

## Fix

실행한 수정 명령:

```bash
cd /home/tim/work/english-90day-android
npx expo install expo-asset
```

변경 파일:

```text
/home/tim/work/english-90day-android/package.json
/home/tim/work/english-90day-android/package-lock.json
```

추가된 dependency:

```json
"expo-asset": "~11.0.5"
```

## Verification

### Resolve 확인

```bash
node -e "console.log(require.resolve('expo-asset/tools/hashAssetFiles'))"
```

결과:

```text
/home/tim/work/english-90day-android/node_modules/expo-asset/tools/hashAssetFiles.js
```

### Unit test

```bash
npm test
```

결과:

```text
# tests 9
# pass 9
# fail 0
```

### TypeScript

```bash
npx tsc --noEmit
```

결과: 성공

### Expo doctor

```bash
npx expo-doctor
```

결과:

```text
18/18 checks passed. No issues detected!
```

### Expo start smoke test

```bash
npx expo start --port 8082 --clear --non-interactive
```

결과:

```text
› Metro waiting on exp://192.168.240.108:8082
```

검증용 Expo 프로세스는 종료했다.

## Current Run Command

다시 실행:

```bash
cd /home/tim/work/english-90day-android
npm start
```

8081 포트가 사용 중이면 Expo prompt에서 `yes`를 선택해 8082를 사용하면 된다.

## Notes

`npm install` 과정에서 audit warning이 남아 있다.

```text
17 vulnerabilities (5 moderate, 12 high)
```

하지만 `npm audit fix --force`는 Expo/React Native dependency 버전을 깨뜨릴 수 있으므로 자동 실행하지 않았다. 필요 시 별도 dependency upgrade 작업으로 다루는 것이 안전하다.

## Related

- [[Index]]
- [[Home]]
