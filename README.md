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
- Review Queue for structurally valid questions that still need an answer review.

## Run

Open `index.html` in a modern browser. The app stores the Question Bank, progress, topics, flashcard status and Review Queue in browser `localStorage`.

> Clearing browser/site data also clears locally stored MedQuiz data. Export important question banks before clearing storage.

## Import: recommended workflow

For reliable production use, **JSON is the canonical input**. PDF/TXT/MD should be treated as source material that is converted or parsed into the canonical JSON format before long-term use.

Recommended schema:

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

### Important import behavior / known issues

> **If an upload appears to be missing questions, do not immediately assume the JSON file is incomplete.** The most common cause is importing a corrected/full dataset with `Gộp` while an older partial copy is already stored in `localStorage`. Use **`Thay thế dataset cùng ID`** for a clean replacement, then compare the import counters before committing.

1. **Use `Thay thế dataset cùng ID` when re-importing a corrected or complete version of the same dataset.**  
   `Gộp` intentionally skips duplicates. If an older partial dataset already exists, Merge mode can make a new upload appear to be missing questions.

2. **Check all import counters before committing:**
   - `Theo metadata`: `dataset.question_count` declared by the file.
   - `Nhận diện`: number of question records actually found.
   - `Sẵn sàng`: valid questions that can enter the Question Bank.
   - `Cần review`: structurally valid questions without a sufficiently resolved answer.
   - `Trùng`: questions skipped by duplicate detection.
   - `Lỗi schema`: malformed records or invalid `correct_answer` references.

   For a clean production `quiz.input.v3` file, the ideal state is that metadata, detected and ready counts match, with zero schema errors.

3. **`Source Archive V3` is intentionally different from production Quiz JSON.**  
   When importing `quiz.source-archive.v3`:
   - questions marked `verified_from_source` go into the Question Bank;
   - unresolved questions go into the Review Queue;
   - unresolved records are not silently treated as wrong schema.

4. **A question is not production-ready unless `correct_answer` points to an existing option ID.**  
   The app does not guess a missing answer when importing canonical Quiz JSON.

5. **Raw PDF import depends on PDF.js loaded from CDN.**  
   PDF import therefore requires network access when using this build. JSON/TXT/MD do not depend on PDF.js extraction. For large or important datasets, convert PDF to reviewed JSON first instead of relying on browser-side PDF parsing as the source of truth.

6. **Duplicate detection is deliberate.**  
   Repeated questions may be skipped based on normalized question text and options. If the uploaded file says it contains more questions than the app imports, inspect the `Trùng`, `Cần review` and `Lỗi schema` counters before assuming the file is incomplete.

7. **Browser storage has practical limits.**  
   Very large datasets may eventually hit `localStorage` quota depending on browser/device. A future production architecture should move the Question Bank and progress history to IndexedDB or a backend database.

## Data philosophy

MedQuiz Pink keeps the Question Bank as the single source of truth. Flashcards reuse the same question records instead of maintaining a second independent content store. This prevents edited quiz questions and flashcards from drifting out of sync.

For imported content, source-derived answer keys and expert-reviewed answers should remain distinguishable in upstream datasets/audit files rather than being silently conflated.
