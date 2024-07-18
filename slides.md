---
theme: default
background: https://cover.sli.dev
title: Elasticsearch🎈 & Slack bot🤖
class: text-center
highlighter: shiki
transition: slide-left
mdc: true
page: 1
---
# Elasticsearch 의도 분석 이해

<p style="position:absolute;right:8em">(12 min)</p>



---
transition: fade-out
page: 2
---
# 개발 과정 소개

### 의도 추출 방법
- 키워드 기반 (BM25) 우선 시도 
- 사용자 질의 키워드가 유사질의에 많이 매칭되는 문서를 찾음

<br>

### 날씨 의도 정의
- 의도 : 날씨 > 오늘/내일/어제/이번주/저번주/어제/그저께/모레
- 유사질의 예) 날씨 > 오늘 : "날씨", "오늘 날씨 어때?", "현재 날씨 알려줘", "현재 기온" ... 

<br>

### 필요 개체명 정의 
- 기상청 단기예보 API 사용 
- @지역(시,군,구...), @날짜, @시간 필요

---
transition: slide-up
page: 4
---
# 의도 분석 방법

### ES 색인
- 노리 사전에 필요한 개체명 사전 등록  ex) 서울시, 강남, 강서구, 오늘, 내일, 한시 등등 
- ES index 에 의도 <-> 유사질의 쌍 색인
```json
{
    "intent" : "날씨 > 다음주",
    "sentences" : [
      {
        "text" : "다음주 날씨"
      },
      {
        "text" : "다음주 날씨 알려줘"
      },
      {
        "text" : "다음주 날씨 어때"
      }, 
      ...
```

---
transition: slide-up
page: 5
---
# 의도 분석 방법

### ES 질의
- 사용자 질의 형태소 분석 후, 등록된 유사질의를 만족하는 의도 검색
- "강남 날씨 내일은 어때?" -> "강남", "**날씨**", "**내일**", "은", "**어떻**", "**어**"
```json
{
  "intent": "날씨 > 내일",
  "sentences": [
    >> {"text": "내일 날씨 어때?"},
    {"text": "내일 날씨가 어떻지?"},
    {"text": "내일 날씨 알려줘."}
  ... 
```

---
transition: slide-up
page: 6
---
# 엔티티 추출 방법

### 엔티티 추출
- 형태소 분석 -> 명사추출 후 기 정의된 태그들과 매칭
  - @timePre : 오전,오후,AM,PM,자정,정오 ..
  - @time : 0시~24시, 영시~스물네시, 십삼시~이십삼시 .. 
  - @si : 서울시.. @gu : 강남구, 강남, 역삼 .., @dong : 역삼동, 역삼1동, 역삼2동 ..

<br>

- 서울시 강남의 내일 오전 날씨는
```json
{
  "parsedEntity": {
    "@time": ["12시"],
    "@si": ["서울시"],
    "@gu": ["강남"],
    "@time_pre": ["오전"]
  }
}
```

---
transition: slide-up
page: 7
---
# 한계점

### 주요 키워드 중복 시, 의도 파악 난해
- "나는 오늘 인천인데 내일 서울의 날씨는 어때?"
  - 의도 분석
    - 검색된 의도 : 날씨 > 오늘, 날씨> 내일
    - 실제 의도 : 내일의 서울 날씨
  - 개체명 선택
    - 인천, 서울 중 지역 선택이 애매함
    - 대화엔진 API 연동 사용 시, 다중 개체명값 처리 방식

---
transition: fade-out
page: 8
---
# Slackbot
[https://api.slack.com](https://api.slack.com) 의 **`Your Apps`** 메뉴 참조

---
transition: slide-up
page: 9
---
# 의도 요청 API
Typescript 라서 node.js 의 기본 api 중 `fetch` 사용

```ts {0|4|5|all}
// src/load.ts
export async function intent(sentence: string) {
  const urlEncoded = encodeURIComponent(sentence)
  return await fetch('.../intent/query'
    + `?sentence=${urlEncoded}`
  )
  .then(resp => resp.json())
}
```

<span v-mark.underline.orange>`"내일 저녁에 부평에서 축구할거야"`</span> 라는 문장의 결과는 
```json {0|8|8-10|4-5|6-7|all}
{
  "responseTime": 8,
  "intentId": "2",
  "intentName": "날씨 > 내일",
  "date": "내일",
  "timePre": ["저녁"],
  "time": null,
  "location": "인천광역시 > 부평구 > 부평1동",
  "nx": 55,
  "ny": 125,
  "parsedEntity": ...
}
```



---
transition: fade-out
layout: image-right
image: https://dev.ccaas.langsa.ai/slack.png
page: 9
---
# 꼴

Slack bot(Slash Commands)로 <span v-mark.underline.blue>`/날씨 쫑알쫑알`</span> 하면 <span v-mark.circle.red>`https://.../weather`</span> 로 요청
```ts {0|5-9|all}
app.use('/weather', async (req, res) => {
  const {user_id: cacheKey, text} = req.body
  const intented: Intent = await intent(text)
  const cached = caches.get(cacheKey)

  checkDate(intented, cached) // "날짜" 의도 확인
  checkTime(intented, cached) // "시간" 의도 확인

  if (unknownLocation(intented)) { // "지역" 의도 확인
    res.json(retry(intented, cacheKey))
  } else {
    res.json(
        await serve(intented, await load(intented), cacheKey)
    )
  }
})
```

---
transition: fade-out
layout: center
page: 10
---
# 끝