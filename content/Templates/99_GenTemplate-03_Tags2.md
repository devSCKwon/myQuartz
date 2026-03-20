---
draft: "true"
---
<%*
// --- 설정 (반드시 수정하세요) ---
const GEMINI_API_KEY = "AIzaSyCCoze6KsMZOIyDQ75tkw63Q_ug8vzi7Tk"; 
const GEMINI_MODEL = "gemini-flash-lite-latest";

// --- 프롬프트 설정 ---
const TAG_PROMPT = `당신은 정보 아키텍처 전문가입니다. 문서를 분석하여 핵심 태그를 생성하세요.
규칙:
1. 결과값은 오직 콤마(,)로 구분된 태그들만 출력하세요. (예: 태그1, 태그2, 태그3)
2. # 기호는 넣지 마세요.
3. 3~5개의 핵심 키워드를 추출하세요.`;

// --- 실행 로직 ---
const currentFile = tp.config.target_file;
const currentContent = tp.file.content;

if (!currentContent) {
    new Notice("문서 내용이 비어있어 태그를 생성할 수 없습니다.");
    return;
}

try {
    // 1. Gemini API 호출 (Native 엔드포인트)
    const url = `https://generativelanguage.googleapis.com/v1beta/models/${GEMINI_MODEL}:generateContent?key=${GEMINI_API_KEY}`;
    
    const response = await tp.obsidian.requestUrl({
        url: url,
        method: "POST",
        contentType: "application/json",
        body: JSON.stringify({
            contents: [{
                parts: [{ text: `${TAG_PROMPT}\n\n문서 내용:\n${currentContent}` }]
            }]
        })
    });

    // 2. 응답 데이터 파싱
    const resultText = response.json.candidates[0].content.parts[0].text;
    
    // 3. 태그 정제 (콤마 구분 및 공백 제거)
    const newTags = resultText.split(",")
        .map(tag => tag.trim())
        .filter(tag => tag.length > 0);

    // 4. Obsidian Frontmatter 업데이트
    await app.fileManager.processFrontMatter(currentFile, (fm) => {
        const existingTags = fm.tags || [];
        // 기존 태그와 새 태그 합치기 및 중복 제거
        fm.tags = [...new Set([...existingTags, ...newTags])];
    });

    new Notice("✅ 태그 생성이 성공적으로 완료되었습니다!");

} catch (e) {
    console.error("Gemini API Error:", e);
    new Notice("❌ 태그 생성 실패: " + e.message);
}
%>