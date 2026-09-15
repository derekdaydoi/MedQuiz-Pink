# MedQuiz Pink

A lightweight browser-based app for revising medical knowledge with quizzes, flashcards, topic tracking, wrong-answer review and importable question banks.

**Copyright hoangderek**

## Current build

The current UI uses the pink medical-study dashboard with semantic status cards:

- **Tổng câu hỏi** → document icon · pink
- **Đã làm** → graduation-cap icon · green
- **Đang đúng** → check-circle icon · teal
- **Cần ôn lại** → review/clock icon · orange
- **Chưa làm** → open-book icon · violet

## Features

- Pink medical-study dashboard with semantic status cards.
- Quiz mode and Challenge mode with timer, streak, scoring and power-ups.
- Flashcards generated directly from the Question Bank.
- Topic creation and topic-based study.
- Persistent progress, wrong-answer pool and redemption/retry flow.
- Question editor and JSON export.
- Import from Quiz JSON V3, Source Archive V3, PDF, TXT and Markdown.
- Staged import flow with **Upload → Analyze → Review → Commit**.
- Inline answer review before unresolved questions are allowed into the Question Bank.

## Run

Open `index.html` in a modern browser. The app stores the Question Bank, progress, topics, flashcard status and legacy Review Queue in browser `localStorage`.

> Clearing browser/site data also clears locally stored MedQuiz data. Export important question banks before clearing storage.

## Import V10: Upload → Analyze → Review → Commit

The import pipeline is intentionally conservative. MedQuiz Pink does **not** silently guess an answer when the uploaded source does not provide enough evidence.

### 1. Upload

Supported input:

- `quiz.input.v3` JSON
- `quiz.normalized.v3` JSON
- `quiz.source-archive.v3` JSON
- legacy MedQuiz JSON
- PDF with a readable text layer
- TXT / Markdown

For reliable production use, **reviewed JSON is still the canonical input**. PDF/TXT/MD are source formats that must pass through the parser and review process before entering the Question Bank.

### 2. Analyze

Every detected question is classified into one of three states:

- **Ready** — stem/options are structurally valid and the answer is supported by a strong source signal.
- **Needs Review** — stem/options are valid, but the answer is missing, conflicting or not sufficiently trustworthy.
- **Broken** — the parser cannot recover a valid single-choice question structure, for example missing stem, broken options or malformed records.

The Analyze screen shows:

- `Theo metadata`
- `Nhận diện`
- `Ready`
- `Needs Review`
- `Trùng`
- `Broken`

The parser also preserves source metadata when available, including PDF page and source question number.

### Answer detection

For raw PDF/TXT/Markdown, strong answer signals currently include patterns such as:

- `Đáp án: B`
- `Đáp án đúng: C`
- `Answer: D`
- a single clearly bold-marked option when that formatting survives extraction
- a single explicit check/correct marker

Detection provenance is kept internally with fields such as answer basis, confidence and issues.

**Important:** the standalone browser build does not use model knowledge to solve medical questions. If the source does not provide a sufficiently reliable answer signal, the question goes to **Needs Review**.

### 3. Review

Questions in **Needs Review** appear directly in the import workspace instead of being silently committed.

For each question, the user can:

- select the correct option with a radio button;
- edit the stem;
- edit each option;
- edit the topic;
- inspect source page / source question number when available;
- see why the parser requested review;
- skip a bad question explicitly.

The review workspace is paginated to avoid rendering very large imports all at once.

`Lưu các câu đã chọn` converts reviewed questions to **Ready** and records them as `user_verified` / `user_review`.

### 4. Commit

Commit is locked while any question remains in **Needs Review**.

The user must either:

- choose/fix the answer and save the reviewed question; or
- explicitly skip that question.

Only **Ready** questions are written to the Question Bank. **Broken / Skipped** records are excluded.

This prevents unresolved questions from contaminating the production quiz bank.

## Recommended production JSON

```json
{
  "schema_version": "quiz.input.v3",
  "dataset": {
    "id": "my-medical-dataset",
    "title": "My Medical Quiz",
    "subject": "Medicine",
    "language": "vi",
    "version": 3,
    "question_count": 1
  },
  "questions": [
    {
      "id": "Q-0001",
      "order": 1,
      "type": "single_choice",
      "subject": "Medicine",
      "topic_id": "TOPIC",
      "topic": "Chủ đề",
      "subtopic_id": "SUBTOPIC",
      "subtopic": "Chủ đề con",
      "stem": "Nội dung câu hỏi?",
      "options": [
        {"id": "A", "text": "Đáp án A"},
        {"id": "B", "text": "Đáp án B"},
        {"id": "C", "text": "Đáp án C"},
        {"id": "D", "text": "Đáp án D"}
      ],
      "correct_answer": "B",
      "explanation": null
    }
  ]
}
```

## Important import behavior / known issues

1. **Use `Thay thế dataset cùng ID` when importing a corrected/full version of the same dataset.**  
   `Gộp` intentionally skips duplicate questions. If an older partial copy already exists in `localStorage`, Merge mode can make a new upload appear to be missing records.

2. **Do not interpret `Broken` as “wrong answer”.**  
   `Broken` means the question structure itself cannot safely enter a single-choice quiz. `Needs Review` is the state for structurally valid questions whose answer still requires confirmation.

3. **`Source Archive V3` unresolved records now enter the staged Review workspace.**  
   They are no longer automatically treated as production-ready questions. A user must resolve or explicitly skip them before Commit.

4. **A canonical JSON question with a missing/invalid `correct_answer` is reviewable when its stem/options are otherwise valid.**  
   It is no longer automatically discarded as a schema error.

5. **PDF import requires PDF.js from CDN.**  
   Network access is required for browser-side PDF extraction in this build.

6. **Scanned/image-only PDFs are not OCR'd by this standalone build.**  
   If the PDF contains almost no text layer, MedQuiz Pink stops the import and asks for OCR or a JSON/TXT source rather than pretending extraction succeeded.

7. **PDF formatting is lossy.**  
   Bold/highlight information may or may not survive PDF extraction depending on how the PDF was generated. Explicit answer-key text is more reliable than visual formatting.

8. **Duplicate detection is deliberate.**  
   Repeated questions are detected using normalized stem + options. Check the `Trùng` counter if the detected count is larger than the final Ready count.

9. **Browser storage has practical limits.**  
   Large question banks can eventually hit `localStorage` quota depending on browser/device. A future production architecture should migrate questions and progress to IndexedDB or a backend database.

## Data philosophy

MedQuiz Pink keeps the Question Bank as the single source of truth. Flashcards reuse the same question records instead of maintaining a second independent content store, preventing edited quiz questions and flashcards from drifting out of sync.

For imports, provenance matters: source-verified answers, user-reviewed answers and unresolved source records should remain distinguishable rather than being silently conflated.
