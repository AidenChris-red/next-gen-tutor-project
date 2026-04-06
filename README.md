## 🧠 핵심 로직: Multi-modal AI Tutor Engine

본 프로젝트는 단순한 텍스트 답변을 넘어, 학생이 업로드한 **이미지(수식, 그래프, 도표)**를 실시간으로 분석하여 단계별 학습 가이드를 제공합니다.

"""
🎓 AI Tutor Engine v2  ·  Grok × Gemini Edition
────────────────────────────────────────────────────────────────────
역할 분담:
  🟡 Grok    → 문제 풀이 / 수식 계산 / PPT 예제 해설 삽입
  🔵 Gemini  → 개념 설명 / 비교표 / 차트 데이터 / 시각화 마크다운

파이프라인:
  1. solve_with_grok()        이미지+텍스트 → 단계별 수식 풀이
  2. explain_with_gemini()    키워드        → 개념 설명 + 표/차트
  3. transcribe_lecture()     음성 파일     → Whisper 스크립트
  4. generate_lecture_ppt()   스크립트      → NotebookLM형 PPT
  5. annotate_professor_ppt() 교수 PPT      → 예제 해설 슬라이드 삽입
────────────────────────────────────────────────────────────────────
"""

import os, base64, json, re
from pathlib import Path

import openai
import google.generativeai as genai
import whisper
from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor


# ══════════════════════════════════════════════════════════
# 0. 클라이언트 초기화
# ══════════════════════════════════════════════════════════

grok_client = openai.OpenAI(
    api_key=os.environ["XAI_API_KEY"],
    base_url="https://api.x.ai/v1",
)
GROK_MODEL = "grok-2-vision-1212"

genai.configure(api_key=os.environ["GOOGLE_API_KEY"])
GEMINI_MODEL = "gemini-2.0-flash-exp"


# ══════════════════════════════════════════════════════════
# 1. 🟡 GROK  ─  문제 풀이 & 수식 계산
# ══════════════════════════════════════════════════════════

GROK_SYSTEM = """
당신은 이공계 전공 문제 풀이 전문 AI입니다.

[필수 규칙]
1. 수식은 반드시 LaTeX로 감싸세요.
   인라인: $E = mc^2$
   블록:   $$\\int_0^\\infty e^{-x}\\,dx = 1$$
2. 풀이 구조:
   ### 📌 핵심 개념
   ### 🔢 풀이 (단계별, 절대 생략 없이)
   ### ✅ 최종 답
   ### 💡 풀이 포인트
3. 중간 계산 단계를 절대 생략하지 마세요.
4. 한국어로 답변하세요.
"""

def _encode_image(path: str) -> tuple[str, str]:
    ext  = Path(path).suffix.lower().lstrip(".")
    mime = {"jpg":"image/jpeg","jpeg":"image/jpeg",
            "png":"image/png","gif":"image/gif"}.get(ext, "image/jpeg")
    with open(path, "rb") as f:
        return base64.b64encode(f.read()).decode(), mime


def solve_with_grok(question: str, image_path: str | None = None) -> str:
    """Grok Vision으로 문제 풀이. image_path 있으면 이미지도 분석."""
    content: list = [{"type": "text", "text": question}]
    if image_path:
        b64, mime = _encode_image(image_path)
        content.append({"type":"image_url","image_url":{"url":f"data:{mime};base64,{b64}"}})

    resp = grok_client.chat.completions.create(
        model=GROK_MODEL,
        messages=[
            {"role":"system","content":GROK_SYSTEM},
            {"role":"user","content":content},
        ],
        temperature=0.2,
        max_tokens=2500,
    )
    return resp.choices[0].message.content


# ══════════════════════════════════════════════════════════
# 2. 🔵 GEMINI  ─  개념 설명 + 비교표 + 차트 JSON
# ══════════════════════════════════════════════════════════

GEMINI_CONCEPT_PROMPT = """
당신은 대학교 이공계 개념 설명 전문 AI입니다.

## 🔵 개념 설명: {keyword}

### 1️⃣ 한 줄 정의
(핵심을 한 문장으로)

### 2️⃣ 직관적 이해
(비유나 실생활 예시)

### 3️⃣ 수식 & 정의
(LaTeX: $$...$$)

### 4️⃣ 비교표
| 구분 | 항목A | 항목B |
|------|-------|-------|
(관련 개념 비교를 Markdown 표로 — 반드시 포함)

### 5️⃣ 차트 데이터 (JSON)
```json
{{
  "type": "bar",
  "title": "...",
  "labels": [...],
  "datasets": [{{"label":"...","data":[...]}}]
}}
```

### 6️⃣ 학습 순서
선수지식 → 현재 개념 → 다음 단계

한국어로 작성하고 표와 차트 데이터는 반드시 포함하세요.
"""

def explain_with_gemini(keyword: str, context: str = "") -> dict:
    """
    Gemini로 개념 설명 + 비교표 + 차트 JSON 반환.
    Returns: {"markdown": str, "chart_json": dict|None}
    """
    model  = genai.GenerativeModel(GEMINI_MODEL)
    prompt = GEMINI_CONCEPT_PROMPT.format(keyword=keyword)
    if context:
        prompt += f"\n\n[추가 맥락]\n{context}"

    text = model.generate_content(prompt).text
    chart_json = None
    m = re.search(r'```json\s*(\{.*?\})\s*```', text, re.DOTALL)
    if m:
        try: chart_json = json.loads(m.group(1))
        except: pass

    return {"markdown": text, "chart_json": chart_json}


def explain_with_gemini_image(image_path: str, question: str) -> dict:
    """이미지(그래프·도표) → Gemini 시각적 개념 설명."""
    model = genai.GenerativeModel(GEMINI_MODEL)
    with open(image_path, "rb") as f:
        img_bytes = f.read()
    ext  = Path(image_path).suffix.lower().lstrip(".")
    mime = {"jpg":"image/jpeg","jpeg":"image/jpeg","png":"image/png"}.get(ext,"image/jpeg")

    text = model.generate_content([
        {"mime_type": mime, "data": img_bytes},
        f"이 이미지의 개념을 설명해주세요.\n{question}\n비교표와 차트 데이터(JSON)를 반드시 포함하세요.",
    ]).text

    chart_json = None
    m = re.search(r'```json\s*(\{.*?\})\s*```', text, re.DOTALL)
    if m:
        try: chart_json = json.loads(m.group(1))
        except: pass

    return {"markdown": text, "chart_json": chart_json}


# ══════════════════════════════════════════════════════════
# 3. 강의 음성 → Whisper 스크립트
# ══════════════════════════════════════════════════════════

def transcribe_lecture(audio_path: str, language: str = "ko") -> dict:
    model  = whisper.load_model("large-v3")
    result = model.transcribe(audio_path, language=language, verbose=False, fp16=False)
    print(f"[Whisper] 변환 완료 | {len(result['text'])}자")
    return result


# ══════════════════════════════════════════════════════════
# 4. 강의 스크립트 → PPT  (Gemini 아웃라인)
# ══════════════════════════════════════════════════════════

THEME = {
    "bg":     RGBColor(0x0F,0x0F,0x23),
    "title":  RGBColor(0x7C,0xD1,0xF0),
    "body":   RGBColor(0xE8,0xE8,0xF0),
    "accent": RGBColor(0xFF,0xC8,0x57),
    "gemini": RGBColor(0x4A,0xB7,0xFF),
    "grok":   RGBColor(0xFF,0xC8,0x57),
}

def _gemini_outline(transcript: str) -> list[dict]:
    model  = genai.GenerativeModel(GEMINI_MODEL)
    prompt = (
        "다음 강의 스크립트를 PPT 슬라이드 아웃라인 JSON으로 변환하세요.\n"
        '형식: [{"title":"...", "bullets":["...","..."]}]\n'
        "슬라이드 수: 8~14장. JSON만 출력.\n\n"
        f"[스크립트]\n{transcript[:5000]}"
    )
    text = model.generate_content(prompt).text.strip()
    m = re.search(r'\[.*\]', text, re.DOTALL)
    if m:
        try: return json.loads(m.group())
        except: pass
    return [{"title":"강의 내용","bullets":[text[:300]]}]


def _add_slide(prs, title, bullets, badge=""):
    layout = prs.slide_layouts[6]
    slide  = prs.slides.add_slide(layout)
    W, H   = prs.slide_width, prs.slide_height
    slide.background.fill.solid()
    slide.background.fill.fore_color.rgb = THEME["bg"]

    if badge:
        b = slide.shapes.add_textbox(Inches(0.4),Inches(0.18),Inches(2.5),Inches(0.38))
        r = b.text_frame.paragraphs[0].add_run()
        r.text = badge; r.font.size = Pt(11); r.font.bold = True
        r.font.color.rgb = THEME["gemini"] if "Gemini" in badge else THEME["grok"]

    tf = slide.shapes.add_textbox(Inches(0.4),Inches(0.6),W-Inches(0.8),Inches(1.1)).text_frame
    run = tf.paragraphs[0].add_run()
    run.text = title; run.font.size = Pt(30); run.font.bold = True
    run.font.color.rgb = THEME["title"]

    sep = slide.shapes.add_shape(1,Inches(0.4),Inches(1.68),W-Inches(0.8),Emu(35000))
    sep.fill.solid(); sep.fill.fore_color.rgb = THEME["accent"]
    sep.line.fill.background()

    body = slide.shapes.add_textbox(Inches(0.4),Inches(1.82),W-Inches(0.8),H-Inches(2.3)).text_frame
    body.word_wrap = True
    for b in bullets:
        p = body.add_paragraph()
        p.text = f"• {b}"; p.font.size = Pt(17)
        p.font.color.rgb = THEME["body"]; p.space_after = Pt(7)


def generate_lecture_ppt(audio_path: str, out: str = "lecture_notes.pptx") -> str:
    print("🎙️  [1/3] Whisper 음성 변환 중...")
    transcript = transcribe_lecture(audio_path)["text"]
    print("🔵 [2/3] Gemini 아웃라인 생성 중...")
    outline = _gemini_outline(transcript)
    print("📊 [3/3] PPT 생성 중...")
    prs = Presentation()
    prs.slide_width = Inches(13.33); prs.slide_height = Inches(7.5)
    _add_slide(prs,"📚 AI 강의 노트",[f"슬라이드 {len(outline)}장  ·  Whisper + Gemini 자동 생성"])
    for s in outline:
        _add_slide(prs,s["title"],s.get("bullets",[]),badge="🔵 Gemini")
    prs.save(out)
    print(f"✅ 저장: {out}")
    return out


# ══════════════════════════════════════════════════════════
# 5. 교수님 PPT → Grok 예제 풀이 → 해설 슬라이드 삽입
# ══════════════════════════════════════════════════════════

KEYWORDS = ["예제","문제","example","exercise","q.","연습","풀어","구하라","계산"]

def _find_problems(pptx_path: str) -> list[dict]:
    prs = Presentation(pptx_path)
    found = []
    for i, slide in enumerate(prs.slides):
        texts = [p.text for sh in slide.shapes if sh.has_text_frame
                 for p in sh.text_frame.paragraphs]
        full = "\n".join(texts)
        if any(k in full.lower() for k in KEYWORDS):
            found.append({"idx":i,"title":texts[0] if texts else f"슬라이드{i+1}","body":full})
    print(f"[PPT] 예제 {len(found)}개 발견")
    return found


def _insert_solution(prs, after_idx, title, solution):
    layout = prs.slide_layouts[6]
    slide  = prs.slides.add_slide(layout)
    W, H   = prs.slide_width, prs.slide_height
    slide.background.fill.solid()
    slide.background.fill.fore_color.rgb = RGBColor(0x0A,0x1A,0x2E)

    b = slide.shapes.add_textbox(Inches(0.4),Inches(0.18),Inches(3),Inches(0.38))
    r = b.text_frame.paragraphs[0].add_run()
    r.text = "🟡 Grok AI 해설"; r.font.size = Pt(12); r.font.bold = True
    r.font.color.rgb = THEME["grok"]

    tf = slide.shapes.add_textbox(Inches(0.4),Inches(0.62),W-Inches(0.8),Inches(1)).text_frame
    run = tf.paragraphs[0].add_run()
    run.text = f"[해설] {title}"; run.font.size = Pt(23); run.font.bold = True
    run.font.color.rgb = THEME["title"]

    body = slide.shapes.add_textbox(Inches(0.4),Inches(1.75),W-Inches(0.8),H-Inches(2.2)).text_frame
    body.word_wrap = True
    display = solution[:1700] + ("\n\n… 상세 풀이 별도 첨부" if len(solution)>1700 else "")
    for line in display.split("\n"):
        if not line.strip(): continue
        p = body.add_paragraph()
        p.text = line; p.font.size = Pt(14); p.font.color.rgb = THEME["body"]

    lst  = prs.slides._sldIdLst
    node = lst[-1]; lst.remove(node); lst.insert(after_idx+2, node)


def annotate_professor_ppt(pptx_path: str, out: str = "annotated.pptx") -> str:
    problems = _find_problems(pptx_path)
    if not problems:
        print("⚠️  예제 슬라이드를 찾지 못했습니다."); return pptx_path
    prs = Presentation(pptx_path)
    for prob in reversed(problems):
        print(f"\n🟡 Grok 풀이 중: {prob['title'][:45]}...")
        sol = solve_with_grok(f"다음 예제 문제를 완전히 풀어주세요.\n\n{prob['body']}")
        _insert_solution(prs, prob["idx"], prob["title"], sol)
    prs.save(out)
    print(f"\n✅ 해설 PPT 저장: {out}  ({len(problems)}개 삽입)")
    return out


# ══════════════════════════════════════════════════════════
# 6. CLI 진입점
# ══════════════════════════════════════════════════════════

if __name__ == "__main__":
    import sys
    cmd = sys.argv[1] if len(sys.argv) > 1 else "help"

    if cmd == "solve":
        q   = sys.argv[2] if len(sys.argv) > 2 else "이 문제를 풀어주세요"
        img = sys.argv[3] if len(sys.argv) > 3 else None
        print(solve_with_grok(q, img))

    elif cmd == "explain":
        result = explain_with_gemini(sys.argv[2])
        print(result["markdown"])
        if result["chart_json"]:
            print("\n[차트 JSON]\n", json.dumps(result["chart_json"], ensure_ascii=False, indent=2))

    elif cmd == "lecture":
        generate_lecture_ppt(sys.argv[2])

    elif cmd == "annotate":
        annotate_professor_ppt(sys.argv[2])

    else:
        print("""
사용법:
  python main.py solve    "질문" [이미지]   🟡 Grok  문제 풀이
  python main.py explain  "키워드"          🔵 Gemini 개념 설명+표+차트
  python main.py lecture  <음성파일>        강의 PPT 자동 생성
  python main.py annotate <pptx파일>        교수님 PPT 해설 삽입

  """
📚 PDF 교재 분석 엔진
────────────────────────────────────────────────
기능:
  1. parse_textbook()     PDF → 전체 텍스트 + 챕터 목록 추출
  2. extract_range()      챕터 or 페이지 범위 → 해당 텍스트 슬라이싱
  3. build_context()      범위 텍스트 → Grok/Gemini 컨텍스트 문자열
  4. save_session()       시험 범위 세션 저장/로드 (JSON)
────────────────────────────────────────────────
"""

import re, json, os
from dataclasses import dataclass, asdict
from pathlib import Path
import pdfplumber

# ──────────────────────────────────────────────
# 데이터 구조
# ──────────────────────────────────────────────

@dataclass
class Chapter:
    index:      int          # 1-based 챕터 번호
    title:      str          # 챕터 제목
    page_start: int          # 시작 페이지 (1-based)
    page_end:   int          # 끝 페이지  (1-based, inclusive)
    text:       str = ""     # 추출된 텍스트 (populate_text 이후 채워짐)

@dataclass
class TextbookSession:
    filename:      str
    total_pages:   int
    chapters:      list[dict]          # Chapter를 dict로 저장
    exam_chapters: list[int] = None    # 선택된 챕터 index 목록
    exam_page_start: int = 1
    exam_page_end:   int = 0           # 0이면 total_pages 사용


# ──────────────────────────────────────────────
# 1. PDF 파싱 & 챕터 추출
# ──────────────────────────────────────────────

# 챕터 헤더 패턴 (한국어/영어 혼용 대학 교재 대응)
CHAPTER_PATTERNS = [
    r'^(Chapter|CHAPTER)\s+(\d+)[:\s]+(.*)',
    r'^(제\s*\d+\s*[장절편])\s*(.*)',
    r'^(\d+)\.\s+([A-Z가-힣].{3,60})$',
    r'^(PART|Part)\s+(\d+)[:\s]+(.*)',
]

def _is_chapter_header(line: str) -> tuple[int | None, str]:
    """
    챕터 헤더 여부 판별.
    Returns (chapter_number, title) or (None, "")
    """
    line = line.strip()
    for pat in CHAPTER_PATTERNS:
        m = re.match(pat, line, re.IGNORECASE)
        if m:
            groups = m.groups()
            # 번호 추출
            num_str = next((g for g in groups if g and re.search(r'\d', g)), None)
            num = int(re.search(r'\d+', num_str).group()) if num_str else None
            title = groups[-1].strip() if groups else line
            return num, title
    return None, ""


def parse_textbook(pdf_path: str, max_pages: int = 0) -> TextbookSession:
    """
    PDF → TextbookSession (챕터 목록 + 페이지 텍스트 포함).
    max_pages: 0이면 전체, 그 외 첫 N페이지만 처리 (대형 교재 속도 조절)
    """
    chapters: list[Chapter] = []
    page_texts: list[str]   = []

    with pdfplumber.open(pdf_path) as pdf:
        total = len(pdf.pages)
        limit = min(total, max_pages) if max_pages else total

        current_ch: Chapter | None = None
        ch_idx = 0

        for i in range(limit):
            page = pdf.pages[i]
            text = page.extract_text() or ""
            page_texts.append(text)

            for line in text.splitlines():
                num, title = _is_chapter_header(line)
                if num is not None and title:
                    # 이전 챕터 마감
                    if current_ch:
                        current_ch.page_end = i   # 현재 페이지 바로 전
                        chapters.append(current_ch)

                    ch_idx += 1
                    current_ch = Chapter(
                        index=ch_idx, title=title,
                        page_start=i + 1, page_end=limit,
                    )

        # 마지막 챕터
        if current_ch:
            current_ch.page_end = limit
            chapters.append(current_ch)

    # 챕터가 감지 안 된 경우 → 25페이지 단위 자동 분할
    if not chapters:
        chunk = max(25, limit // 10)
        ch_idx = 0
        for start in range(0, limit, chunk):
            ch_idx += 1
            end = min(start + chunk, limit)
            # 시작 페이지에서 첫 비어있지 않은 줄을 제목으로
            title_text = ""
            for ln in (page_texts[start] or "").splitlines():
                if ln.strip():
                    title_text = ln.strip()[:60]
                    break
            chapters.append(Chapter(
                index=ch_idx,
                title=title_text or f"섹션 {ch_idx}",
                page_start=start + 1,
                page_end=end,
            ))

    # 챕터별 텍스트 채우기
    for ch in chapters:
        s, e = ch.page_start - 1, ch.page_end
        ch.text = "\n".join(page_texts[s:e])

    session = TextbookSession(
        filename=Path(pdf_path).name,
        total_pages=total,
        chapters=[asdict(c) for c in chapters],
        exam_page_end=total,
    )
    return session


# ──────────────────────────────────────────────
# 2. 시험 범위 텍스트 추출
# ──────────────────────────────────────────────

def extract_range(session: TextbookSession, max_chars: int = 12000) -> str:
    """
    선택된 챕터 or 페이지 범위의 텍스트를 반환.
    max_chars: Grok/Gemini 컨텍스트 한도 (토큰 절약)
    """
    selected_chapters = session.exam_chapters or []
    chapters = session.chapters

    if selected_chapters:
        texts = []
        for ch in chapters:
            if ch["index"] in selected_chapters:
                texts.append(f"=== {ch['title']} ===\n{ch['text']}")
        combined = "\n\n".join(texts)
    else:
        # 페이지 범위로 대신 추출
        p_start = max(0, session.exam_page_start - 1)
        p_end   = min(session.total_pages, session.exam_page_end)
        texts = []
        for ch in chapters:
            cs, ce = ch["page_start"] - 1, ch["page_end"]
            if cs < p_end and ce > p_start:
                texts.append(ch["text"])
        combined = "\n\n".join(texts)

    # 길이 제한
    if len(combined) > max_chars:
        combined = combined[:max_chars] + "\n\n[... 교재 내용 일부 생략 ...]"

    return combined


def build_context(session: TextbookSession) -> str:
    """AI 프롬프트에 주입할 교재 컨텍스트 블록 생성"""
    range_text = extract_range(session)
    chapter_names = []
    for ch in session.chapters:
        if session.exam_chapters and ch["index"] in session.exam_chapters:
            chapter_names.append(f"Ch.{ch['index']} {ch['title']}")

    scope_label = (
        "챕터: " + ", ".join(chapter_names) if chapter_names
        else f"p.{session.exam_page_start}~{session.exam_page_end}"
    )

    return f"""
[📚 교재 참고 자료 — {session.filename}]
[시험 범위: {scope_label}]

{range_text}

위 교재 내용을 반드시 참고하여 답변하세요.
교재에 없는 내용을 추가할 경우 "(교재 외 보충)" 표시를 해주세요.
"""


# ──────────────────────────────────────────────
# 3. 세션 저장 / 로드
# ──────────────────────────────────────────────

SESSION_DIR = "sessions"
os.makedirs(SESSION_DIR, exist_ok=True)

def save_session(session: TextbookSession, session_id: str) -> str:
    path = os.path.join(SESSION_DIR, f"{session_id}.json")
    data = {
        "filename":        session.filename,
        "total_pages":     session.total_pages,
        "chapters":        session.chapters,
        "exam_chapters":   session.exam_chapters,
        "exam_page_start": session.exam_page_start,
        "exam_page_end":   session.exam_page_end,
    }
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    return path


def load_session(session_id: str) -> TextbookSession | None:
    path = os.path.join(SESSION_DIR, f"{session_id}.json")
    if not os.path.exists(path):
        return None
    with open(path, encoding="utf-8") as f:
        data = json.load(f)
    s = TextbookSession(**data)
    return s


def update_exam_range(
    session_id: str,
    exam_chapters: list[int] | None = None,
    page_start: int | None = None,
    page_end:   int | None = None,
) -> TextbookSession | None:
    """시험 범위만 업데이트하고 저장"""
    session = load_session(session_id)
    if not session:
        return None
    if exam_chapters is not None:
        session.exam_chapters = exam_chapters
    if page_start is not None:
        session.exam_page_start = page_start
    if page_end is not None:
        session.exam_page_end = page_end
    save_session(session, session_id)
    return session
""")
