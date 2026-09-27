# KIND 뉴스레터 → 언어별 4K MP4 → 블로그: 올바른 과정 (한 번에 끝나는 순서)

한 호(예: `092626` = 2026년 9월 26일자, 9월 25일 보도 기준)를 만드는 전체 과정이다.
시행착오 없이 이 순서만 따르면 된다.

## 0. 준비물 (한 번만)
| 위치 | 파일 | 내용 |
|---|---|---|
| 작업 서버(`kind/`) | `.env` | `TYPECAST_API_KEY`, `TYPECAST_VOICE_ID`(인화 `tc_6296a815b958a8ed9610b189`), `ELEVENLABS_API_KEY`, `PEXELS_API_KEY`, `PIXABAY_API_KEY` |
| 작업 서버 | `templates_v2/kind_{kr,en,jp}_template.html` | 뉴스레터 템플릿(16:9 화면 27장) |
| 내 PC | `Downloads\kind_<호>\` | `run_all.bat`, `media.ps1`, `tts.ps1`, `env.txt`, `media_plan.csv`, `script_{kr,en,jp}.txt` |

- 설치(한 번): `pip install playwright numpy pillow && playwright install chromium`, `cd fonts && npm install`, `ffmpeg` 필요
- 작업 서버는 Pexels·Pixabay·Wikimedia·Typecast·ElevenLabs에 직접 접속할 수 없다. **4K 소재 받기와 녹음은 PC에서** 한다.
- 키는 `.env`와 PC의 `env.txt`에만 둔다. GitHub에는 올리지 않는다.

## 1. 기사 쓰기 (작업 서버)
`editions/<호>/content_kr.py`에 27쪽 분량을 쓴다.
- 구성: `HERO`, `SIDE02~03`, `NEWS04~10`, `ISSUES` 5개, `EXTRA`(TRADITION·FACTCHECK·HISTORYTODAY·HIDDEN·SPOTLIGHT), `WORLD` 6개
- 모든 항목에 원문 `source_url`을 단다. 의견이 갈리는 사안은 `pro`/`con`으로 양쪽을 함께 싣는다.
- 영어·일본어판(`content_en.py`, `content_jp.py`)은 같은 구조로 번역한다.

## 2. HTML과 내레이션 원고 만들기 (작업 서버)
```bash
for L in kr en jp; do python render_v2.py --edition <호> --lang $L --script-only; done
```
- 결과는 `out/v2/kind_<호>_<lang>.html`과 `out/v2/video/<lang>/..._script.txt`(27쪽 원고)이다.
- 원고는 화면에 보이는 제목과 본문 그대로다. 그래서 자막과 목소리가 늘 일치한다.

## 3. PC로 보낼 파일 (작업 서버 → PC)
`media_plan.csv`에 쪽마다 한 줄을 쓴다. 형식은 `KEY;영상 검색어1|검색어2|…;사진 검색어1|…`이다.
검색어는 보편적이고 밝은 장면으로 고르고, 특정 인물이 드러나지 않게 한다.
원고 3개, `env.txt`와 함께 `Downloads\kind_<호>\`에 넣는다.

## 4. PC에서 `run_all.bat` 더블클릭
1. **media.ps1**: 쪽마다 소재를 하나씩 받는다.
   - **4K 영상**(3840×2160, 8~90초): Pexels Video → Pixabay → Wikimedia Commons 순서로 찾는다.
   - **4K 사진**(가로 3840 이상): Wikimedia Commons → Pexels 순서로 찾는다.
   - 같은 소재는 두 번 쓰지 않는다. 작가, 라이선스, 원본 주소는 `media\credits.csv`에 적는다.
2. **tts.ps1**: 원고를 쪽마다 녹음해 `kr\`, `en\`, `jp\`의 `pNN.mp3`로 저장한다.
   - 한국어는 Typecast 인화, 영어는 ElevenLabs Brian, 일본어는 ElevenLabs Hiro S다.
   - 이미 있는 파일은 건너뛴다. 실패하면 `tts_log.txt`에 이유가 남는다.
3. 창에 `===== done =====`이 뜨면 끝이다.

## 5. 가져와 정리하기 (작업 서버)
PC 폴더의 `media\*`, `kr|en|jp\*.mp3`를 업로드 폴더로 가져온다(한 번에 50개·500MB 이하). 그다음 아래를 실행한다.
```bash
python ingest_v2.py --edition <호> --src <업로드 폴더> --delete-src
```
- 렌더용: 3840×2160, 60초 이하로 맞춘 영상과 사진이 `editions/<호>/media/`에 들어간다.
- 웹용 사본: 1280×720 영상과 1920 사진이 `out/v2/web/<호>/`에 들어간다.
- 내레이션: `editions/<호>/narration/<lang>/`에 들어간다.

## 6. 언어별 4K MP4 만들기 (작업 서버)
```bash
for L in kr en jp; do python render_v2.py --edition <호> --lang $L; done
```
- 글자: HTML을 3840×2160으로 캡처하므로 **대본과 자막은 항상 4K**다.
- 배경: 쪽마다 4K 영상이 움직인다. 영상이 없는 쪽은 4K 사진을 천천히 확대하며 움직인다. 용량을 줄이려고 살짝 흐리고 어둡게 한다.
- 소리: 내레이션에 잔잔한 배경음을 깔고, 음량을 -14 LUFS로 맞춘다. 쪽 길이는 내레이션 길이에 1.2초를 더한 값이다.
- 결과: `out/v2/video/<lang>/`
  - 쪽별 `kind_<호>_<lang>_pNN.mp4`(각 24MB 이하 → GitHub 업로드 가능)
  - 전체 `_long_16x9.mp4`, `.srt` 자막, 대본
- 중간에 끊겨도 다시 실행하면 끝난 쪽은 건너뛴다.

## 7. 블로그 글 만들기 (작업 서버)
```bash
python blog_v2.py --edition <호>
```
결과는 `out/v2/blog/<호>/`에 생긴다.
- 파일마다 `*.html`(미리보기·복사용), `*_paste.html`(HTML 모드 붙여넣기용 본문), `*.meta.json`(제목·슬러그·요약·태그)이 만들어진다.

| 플랫폼 | 언어 | 올리는 법 |
|---|---|---|
| WordPress | 한·영·일 | 새 글 → '사용자 정의 HTML' 블록에 `wordpress_<lang>_paste.html` → 메타의 제목·슬러그·요약·태그 입력 |
| AdSense 블로그(Blogger) | 한·영·일 | 새 글 → HTML 보기에 `adsense_<lang>_paste.html` → '검색 설명'에 `search_description`, 라벨에 태그. 본문 속 `<!-- AdSense … -->` 주석 자리에 게시물 내 광고 삽입 |
| Tistory | 한 | 글쓰기 → HTML 모드에 `tistory_kr_paste.html` → 태그 입력 |
| Naver 블로그 | 한 | `naver_kr.html`을 브라우저로 열기 → 전체 선택·복사 → 스마트에디터에 붙여넣기(사진은 네이버가 자동으로 올림) → 태그 10개 |
| Substack | 한·영·일 | `substack_<lang>.html`을 브라우저로 열기 → 본문 복사 → 새 글에 붙여넣기 → 부제는 `subtitle` |

- 여러 곳에 같은 글을 올릴 때는 WordPress 글을 원본(canonical)으로 정한다. 다른 곳에는 "원문: (WordPress 주소)"를 덧붙이면 검색 중복 문제를 줄일 수 있다.

## 8. 올리기 (작업 서버)
- 영상 저장소 `soulthread88/videos`의 `news-<호>` 브랜치에 다음을 올린다.
  - `news/<호>/longform/<lang>/`: 쪽별 MP4, SRT, 대본
  - `news/<호>/web/`: 웹용 사본
  - `news/<호>/html/`, `news/<호>/blog/`
- 뉴스레터 HTML과 블로그는 `web/` 사본을 쓴다. 따라서 이 브랜치가 올라가 있어야 사진과 영상이 보인다.

## 점검표
- [ ] 27쪽 모두 원문 링크가 있다
- [ ] 쪽별 MP4가 3840×2160이고 오디오(내레이션)가 들어 있으며 24MB 이하다
- [ ] `credits.csv`에 27쪽 영상·사진 출처가 있다(블로그 글 맨 아래에 자동 표기)
- [ ] 키가 GitHub에 올라가지 않았다
