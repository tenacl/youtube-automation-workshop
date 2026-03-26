# YouTube 자동 업로드 — Google Apps Script

> 구글 드라이브 특정 폴더에 영상을 넣으면 자동으로 YouTube에 업로드되는 스크립트.
> Opal에서 생성한 영상 → 드라이브에 저장 → 자동 업로드 파이프라인.

---

## 사전 준비

1. **YouTube Data API v3** 활성화 (이미 했으면 패스)
2. **구글 드라이브에 업로드 폴더 생성** (예: `YouTube-Auto-Upload`)
3. 업로드된 영상을 옮길 **완료 폴더** 생성 (예: `YouTube-Uploaded`)

---

## 설정 방법

### 1단계: Apps Script 프로젝트 생성

1. [script.google.com](https://script.google.com) 접속
2. **새 프로젝트** 클릭
3. 프로젝트 이름: `YouTube Auto Upload`

### 2단계: 코드 붙여넣기

아래 코드를 전체 복사해서 `코드.gs`에 붙여넣기:

```javascript
/**
 * YouTube Auto Upload Script
 * 구글 드라이브 폴더의 영상을 자동으로 YouTube에 업로드합니다.
 * 
 * 설정:
 * 1. UPLOAD_FOLDER_ID: 업로드할 영상이 들어있는 드라이브 폴더 ID
 * 2. DONE_FOLDER_ID: 업로드 완료된 영상을 옮길 폴더 ID
 * 3. DEFAULT_PRIVACY: 기본 공개 설정 (private/unlisted/public)
 */

// ═══════════════════════════════════════
// 설정 — 여기만 수정하세요
// ═══════════════════════════════════════
const UPLOAD_FOLDER_ID = '여기에_업로드폴더_ID';  // 드라이브 폴더 ID
const DONE_FOLDER_ID = '여기에_완료폴더_ID';       // 완료 폴더 ID  
const DEFAULT_PRIVACY = 'private';                 // private / unlisted / public

// ═══════════════════════════════════════
// 메인 함수 — 트리거로 실행됨
// ═══════════════════════════════════════
function autoUploadVideos() {
  const uploadFolder = DriveApp.getFolderById(UPLOAD_FOLDER_ID);
  const doneFolder = DriveApp.getFolderById(DONE_FOLDER_ID);
  const files = uploadFolder.getFiles();
  
  let uploadCount = 0;
  
  while (files.hasNext()) {
    const file = files.next();
    const mimeType = file.getMimeType();
    
    // 영상 파일만 처리
    if (!mimeType.startsWith('video/')) {
      Logger.log(`스킵 (영상 아님): ${file.getName()} (${mimeType})`);
      continue;
    }
    
    try {
      // 파일명에서 메타데이터 추출
      const metadata = parseFileName(file.getName());
      
      Logger.log(`업로드 시작: ${file.getName()}`);
      
      // YouTube 업로드
      const video = YouTube.Videos.insert(
        {
          snippet: {
            title: metadata.title,
            description: metadata.description,
            tags: metadata.tags,
            categoryId: '22'  // People & Blogs (기본값)
          },
          status: {
            privacyStatus: DEFAULT_PRIVACY,
            selfDeclaredMadeForKids: false
          }
        },
        'snippet,status',
        file.getBlob()
      );
      
      Logger.log(`✅ 업로드 완료: ${video.snippet.title}`);
      Logger.log(`   영상 ID: ${video.id}`);
      Logger.log(`   URL: https://youtube.com/watch?v=${video.id}`);
      
      // 완료 폴더로 이동
      file.moveTo(doneFolder);
      uploadCount++;
      
      // 알림 이메일 발송
      sendNotification(video, file.getName());
      
    } catch (error) {
      Logger.log(`❌ 업로드 실패: ${file.getName()}`);
      Logger.log(`   에러: ${error.message}`);
    }
  }
  
  Logger.log(`\n총 ${uploadCount}개 영상 업로드 완료`);
}

// ═══════════════════════════════════════
// 파일명에서 메타데이터 추출
// ═══════════════════════════════════════
// 파일명 형식: "제목__태그1,태그2,태그3.mp4"
// 또는 그냥: "제목.mp4"
function parseFileName(fileName) {
  // 확장자 제거
  const nameWithoutExt = fileName.replace(/\.[^/.]+$/, '');
  
  // __ 로 구분되어 있으면 태그 분리
  const parts = nameWithoutExt.split('__');
  const title = parts[0].trim();
  const tags = parts.length > 1 
    ? parts[1].split(',').map(t => t.trim()) 
    : [title];
  
  const description = `${title}\n\n` +
    `${tags.map(t => '#' + t.replace(/\s/g, '')).join(' ')}\n\n` +
    `---\n이 영상은 자동 업로드되었습니다.`;
  
  return { title, description, tags };
}

// ═══════════════════════════════════════
// 업로드 완료 알림
// ═══════════════════════════════════════
function sendNotification(video, originalFileName) {
  const email = Session.getActiveUser().getEmail();
  const subject = `✅ YouTube 업로드 완료: ${video.snippet.title}`;
  const body = `영상이 업로드되었습니다.\n\n` +
    `제목: ${video.snippet.title}\n` +
    `원본 파일: ${originalFileName}\n` +
    `공개 상태: ${DEFAULT_PRIVACY}\n` +
    `URL: https://youtube.com/watch?v=${video.id}\n\n` +
    `YouTube Studio에서 확인하세요: https://studio.youtube.com/video/${video.id}/edit`;
  
  MailApp.sendEmail(email, subject, body);
}

// ═══════════════════════════════════════
// AI 메타데이터 생성 (선택사항)
// Gemini API로 제목/설명/태그 자동 생성
// ═══════════════════════════════════════
function generateMetadataWithAI(fileName) {
  // Gemini API 키가 있으면 사용
  const apiKey = PropertiesService.getScriptProperties().getProperty('GEMINI_API_KEY');
  if (!apiKey) return null;
  
  const prompt = `다음 영상 파일명을 보고 YouTube용 메타데이터를 JSON으로 만들어줘.
파일명: ${fileName}

JSON 형식:
{
  "title": "SEO 최적화된 제목 (60자 이내)",
  "description": "영상 설명 (해시태그 포함, 200자)",
  "tags": ["태그1", "태그2", "태그3", "태그4", "태그5"]
}

JSON만 반환해줘.`;

  const response = UrlFetchApp.fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`,
    {
      method: 'post',
      contentType: 'application/json',
      payload: JSON.stringify({
        contents: [{ parts: [{ text: prompt }] }]
      })
    }
  );
  
  try {
    const result = JSON.parse(response.getContentText());
    const text = result.candidates[0].content.parts[0].text;
    // JSON 블록 추출
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      return JSON.parse(jsonMatch[0]);
    }
  } catch (e) {
    Logger.log('AI 메타데이터 생성 실패: ' + e.message);
  }
  return null;
}

// ═══════════════════════════════════════
// 테스트 함수
// ═══════════════════════════════════════
function testSetup() {
  try {
    const uploadFolder = DriveApp.getFolderById(UPLOAD_FOLDER_ID);
    const doneFolder = DriveApp.getFolderById(DONE_FOLDER_ID);
    Logger.log(`✅ 업로드 폴더: ${uploadFolder.getName()}`);
    Logger.log(`✅ 완료 폴더: ${doneFolder.getName()}`);
    
    const files = uploadFolder.getFiles();
    let count = 0;
    while (files.hasNext()) {
      const f = files.next();
      Logger.log(`  파일: ${f.getName()} (${f.getMimeType()})`);
      count++;
    }
    Logger.log(`\n총 ${count}개 파일 발견`);
    Logger.log('\n설정이 올바릅니다! autoUploadVideos 함수를 실행하세요.');
  } catch (e) {
    Logger.log(`❌ 설정 오류: ${e.message}`);
    Logger.log('폴더 ID를 확인해주세요.');
  }
}
```

### 3단계: YouTube API 서비스 추가

1. Apps Script 에디터에서 왼쪽 **서비스(+)** 클릭
2. **YouTube Data API v3** 검색 → 추가
3. 식별자: `YouTube` (기본값 유지)

### 4단계: 폴더 ID 설정

1. 구글 드라이브에서 업로드 폴더 열기
2. URL에서 폴더 ID 복사: `drive.google.com/drive/folders/`**여기가_ID**
3. 코드의 `UPLOAD_FOLDER_ID`와 `DONE_FOLDER_ID`에 붙여넣기

### 5단계: 테스트

1. `testSetup` 함수 실행 → 폴더 연결 확인
2. 업로드 폴더에 테스트 영상 넣기
3. `autoUploadVideos` 함수 실행
4. YouTube Studio에서 확인

### 6단계: 자동 실행 설정 (선택)

1. Apps Script에서 **트리거(⏰)** 클릭
2. **트리거 추가**:
   - 함수: `autoUploadVideos`
   - 이벤트 소스: 시간 기반
   - 유형: 분 단위 타이머 → 매 15분
3. 저장

---

## 파일명 규칙

| 형식 | 예시 | 결과 |
|------|------|------|
| `제목.mp4` | `캠핑체어리뷰.mp4` | 제목: 캠핑체어리뷰 |
| `제목__태그1,태그2.mp4` | `캠핑체어리뷰__캠핑,아웃도어,체어.mp4` | 제목 + 태그 자동 설정 |

---

## Opal → 드라이브 → YouTube 전체 흐름

```
1. Opal에서 프로모션 영상 생성
   ↓
2. 영상 다운로드
   ↓  
3. 구글 드라이브 "YouTube-Auto-Upload" 폴더에 저장
   (파일명: "상품명__태그1,태그2.mp4")
   ↓
4. Apps Script가 15분마다 폴더 체크
   ↓
5. 새 영상 발견 → YouTube 자동 업로드 (private)
   ↓
6. 완료 이메일 발송 + 파일 완료 폴더로 이동
   ↓
7. YouTube Studio에서 확인 후 공개 전환
```

---

## AI 메타데이터 생성 (고급)

Gemini API 키가 있으면 파일명만으로 제목/설명/태그를 AI가 자동 생성합니다.

설정 방법:
1. Apps Script → 프로젝트 설정 → 스크립트 속성
2. 속성 추가: `GEMINI_API_KEY` = 여러분의 Gemini API 키

---

## FAQ

**Q. 업로드 용량 제한?**  
Apps Script의 파일 처리 한도는 50MB. 그 이상은 YouTube Studio에서 직접 업로드.

**Q. 실행 시간 제한?**  
Apps Script는 6분 제한. 영상 1~2개씩 올리면 충분. 대량 업로드는 여러 번 실행.

**Q. private으로 올리는 이유?**  
실수 방지. 메타데이터 확인 후 public으로 전환하는 게 안전.
