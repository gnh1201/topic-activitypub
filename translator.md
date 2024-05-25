# LibreTranslate API compatible gateway

A customized translation service that is compatible with the LibreTranslate API

## How to test

### FormData

```bash
curl -X POST -H "Content-Type: application/x-www-form-urlencoded" -d 'q=hello world&source=en&target=ko' http://localhost:5000/translate
```

### JSON
```bash
curl -X POST -H "Content-Type: application/json" -d '{"q":["hello world"],"source":"en","target":"ko"}' http://localhost:5000/translate
```

## Example: hello.py

```py
#!/usr/bin/env python3
#
# hello.py
# serverless translation service gateway
#
# @author: Namhyeon Go <abuse@catswords.net>
# @created_on: 2024-03-15
# @updated_on: 2024-05-25
#

import traceback
from flask import Flask, request, jsonify   # Not supported on the serverless
#from flask import request, jsonify
import translators as ts
import json

app = Flask(__name__)    # not supported on the serverless

# 기본 번역기 지정
translator = 'sysTran'

# 언어 코드와 설명을 별개의 리스트로 분리
language_codes = [
    'af', 'sq', 'am', 'ar', 'hy', 'az', 'eu', 'be', 'bn', 'bs', 'bg', 'ca', 'ceb', 'ny',
    'zh-CN', 'zh-TW', 'co', 'hr', 'cs', 'da', 'nl', 'en', 'eo', 'et', 'tl', 'fi', 'fr',
    'fy', 'gl', 'ka', 'de', 'el', 'gu', 'ht', 'ha', 'haw', 'iw', 'hi', 'hmn', 'hu', 'is',
    'ig', 'id', 'ga', 'it', 'ja', 'jw', 'kn', 'kk', 'km', 'ko', 'ku', 'ky', 'lo', 'la',
    'lv', 'lt', 'lb', 'mk', 'mg', 'ms', 'ml', 'mt', 'mi', 'mr', 'mn', 'my', 'ne', 'no',
    'ps', 'fa', 'pl', 'pt', 'pa', 'ro', 'ru', 'sm', 'gd', 'sr', 'st', 'sn', 'sd', 'si',
    'sk', 'sl', 'so', 'es', 'su', 'sw', 'sv', 'tg', 'ta', 'te', 'th', 'tr', 'uk', 'ur',
    'uz', 'vi', 'cy', 'xh', 'yi', 'yo', 'zu'
]
language_names = [
    '아프리칸스어', '알바니아어', '암하라어', '아랍어', '아르메니아어', '아제르바이잔어', '바스크어', '벨라루스어',
    '벵골어', '보스니아어', '불가리아어', '카탈로니아어', '세부아노어', '치체와어', '중국어(간체)', '중국어(번체)',
    '코르시카어', '크로아티아어', '체코어', '덴마크어', '네덜란드어', '영어', '에스페란토어', '에스토니아어',
    '필리핀어', '핀란드어', '프랑스어', '프리지아어', '갈리시아어', '조지아어', '독일어', '그리스어', '구자라트어',
    '아이티어', '하우사어', '하와이어', '히브리어', '힌디어', '몽어', '헝가리어', '아이슬란드어', '이그보어',
    '인도네시아어', '아일랜드어', '이탈리아어', '일본어', '자바어', '칸나다어', '카자흐어', '크메르어', '한국어',
    '쿠르드어', '키르기스어', '라오어', '라틴어', '라트비아어', '리투아니아어', '룩셈부르크어', '마케도니아어',
    '말라가시어', '말레이어', '말라얄람어', '몰타어', '마오리어', '마라티어', '몽골어', '버마어', '네팔어',
    '노르웨이어', '파슈토어', '페르시아어', '폴란드어', '포르투갈어', '펀잡어', '루마니아어', '러시아어', '사모아어',
    '스코틀랜드 게일어', '세르비아어', '소토어', '쇼나어', '신디어', '싱할라어', '슬로바키아어', '슬로베니아어',
    '소말리아어', '스페인어', '순다어', '스와힐리어', '스웨덴어', '타지크어', '타밀어', '텔루구어', '태국어',
    '터키어', '우크라이나어', '우르두어', '우즈베크어', '베트남어', '웨일스어', '코사어', '이디시어', '요루바어',
    '주앙어', '중국어', '줄루어'
]

# 프록시 지정
#proxies = {
#    'http': 'http://localhost:40000',
#    'https': 'http://localhost:40000'
#}
proxies = {}

# 언어 이름 확인
def get_language_name(code):
    try:
        index = language_codes.index(code)
    except:
        index = -1
    return language_names[index] if index > -1 else code

# 번역 API 엔드포인트
@app.route('/translate', methods=['POST'])    # Not supported on the serverless
def translate():
    # 번역하고자 하는 텍스트
    original_texts = []

    # 클라이언트로부터 요청 받은 데이터 (formData)
    data = request.form.to_dict()
    if 'q' in data:
        original_texts.append(data['q'])

    # 클라이언트로부터 JSON 요청으로 데이터를 받은 경우 (jsonData)
    try:
       jsondata = json.loads(request.data)
       if 'q' in jsondata and 'target' in jsondata:
           data.update({
               "q": jsondata['q'],
               "source": jsondata['source'] if 'source' in jsondata else 'auto',
               "target": jsondata['target']
           })
           for text in data['q']:
               original_texts.append(text)
    except Exception as e:
        traceback.print_exc()
        app.logger.info(str(e))
        #return jsonify({"error": str(e)}), 500

    # 필수 요청 매개변수 확인
    if 'q' not in data or 'target' not in data:
        return jsonify({"error": "Missing required parameters"}), 400

    # 출발지 언어가 없는 경우 자동으로 설정
    from_language = data['source'] if 'source' in data else 'auto'

    # 번역 진행
    try:
        translated_texts = []
        for text in original_texts:
            translated_texts.append(ts.translate_text(text, proxies=proxies, from_language=from_language, to_language=data['target'], translator=translator))
        return jsonify({"translatedText": translated_texts}), 200
    except Exception as e:
        traceback.print_exc()
        app.logger.info(str(e))
        return jsonify({"error": str(e)}), 500

# 지원되는 언어 목록 API 엔드포인트
@app.route('/languages', methods=['GET'])    # Not supported on the serverless
def supported_languages():
    try:
        # translators 패키지가 지원하는 언어 목록 가져오기
        languages = ts.get_languages(translator=translator)

        # 언어 목록을 주어진 형식에 맞게 변환
        formatted_languages = []
        for code in languages:
            formatted_languages.append({
                "code": code,
                "name": get_language_name(code),
                "targets": list(languages.keys())
            })

        return jsonify(formatted_languages), 200
    except Exception as e:
        traceback.print_exc()
        app.logger.info(str(e))
        return jsonify({"error": str(e)}), 500

'''
def main():
    #route = request.path    # Not supported on the serverless
    route = request.args.get('route')   # Alternative to the request.path

    if request.method == 'POST' and route == '/translate':
        response_text, _ = translate()
        return response_text

    if request.method == 'GET' and route == '/languages':
        response_text, _ = supported_languages()
        return response_text

    if request.method == 'GET' and route == '/':
        return jsonify({"welcome": True})

    return jsonify({"welcome": False})

    #app.run(debug=True)    # Not supported on the serverless
'''

if __name__ == "__main__":
    app.run(host='0.0.0.0', debug=True)
```

## language-service.service

```service
[Unit]
Description=serverless translation service gateway
Documentation=https://gist.github.com/gnh1201/93cecb002080da2b65eda3dd6032f05a
After=network.target

[Service]
User=gosomi
Group=gosomi
WorkingDirectory=/home/gosomi/Projects/language-service
ExecStart=/usr/bin/python3 /home/gosomi/Projects/language-service/hello.py
Restart=always

[Install]
WantedBy=multi-user.target
```

## Original article
* https://gist.github.com/gnh1201/93cecb002080da2b65eda3dd6032f05a

## Contact me
* ActivityPub [@gnh1201@catswords.social](https://catswords.social/@gnh1201)
* abuse@catswords.net
