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

### 의도 데이터 정리
- 기상청 데이터 가져오고 어쩌고
- 오늘 내일 어제 다음주 지난주 어제 등등...

<br>
<hr>
<br>

### 암튼 열심히 했음
- 그러나 기상청 단기예보 API 의 한계에서...
- 의도 분석 내용이 축소 돼버렸고...



---
transition: slide-up
page: 3
---
# 의도 분석 내용

### 시간
- 1~24시
- 한시~이십사시 (영시)
- 한시~스물네시 (빵시)

### 시간대
- 아침, 점심, 저녁
- 오전, 오후
- AM, PM



---
transition: fade-out
page: 4
---
# 의도 요청 API
Typescript 라서 node.js 의 기본 api 중 `fetch` 사용

```ts {0|3|all}
// src/load.ts
export async function intent(sentence: string) {
  return await fetch(`.../intent/query?sentence=${encodeURIComponent(sentence)}`)
  .then(resp => resp.json())
}
```

<span v-mark.underline.orange>`"내일 저녁에 부평에서 축구할거야"`</span> 라는 문장의 결과는 
```json {0|8|8-10|4-5|6|all}
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
page: 5
---
# 꼴

Slack bot(Slash Commands)로 <span v-mark.underline.blue>`/날씨 쫑알쫑알`</span> 하면 <span v-mark.circle.red>`https://.../weather`</span> 로 요청
```ts {5-9|all}
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
page: 6
---
# 끝