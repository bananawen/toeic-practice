# TOEIC 練習網站 - 資料庫架構設計

## 概述

本文件定義 TOEIC 練習網站的資料庫架構，旨在支援大量資料儲存與高效能查詢。

---

## 推薦資料庫系統

### 選擇：**PostgreSQL**

**理由：**
- 支援 JSON 類型（靈活儲存題目選項、複雜資料）
- 優秀的擴展性，適合未來大量資料成長
- 強大的索引機制，提升查詢效能
- 支援陣列類型與全文檢索
- 成熟的叢集與複製功能

---

## 資料表結構

### 1. `users` - 使用者資料表

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | BIGSERIAL | 主鍵（自動遞增） |
| `email` | VARCHAR(255) | 電子郵件（唯一） |
| `username` | VARCHAR(50) | 使用者名稱 |
| `password_hash` | VARCHAR(255) | 密碼雜湊 |
| `avatar_url` | VARCHAR(500) | 頭像 URL |
| `target_score` | INT | 目標分數（預設 900） |
| `created_at` | TIMESTAMP | 建立時間 |
| `updated_at` | TIMESTAMP | 更新時間 |
| `last_login_at` | TIMESTAMP | 最後登入時間 |
| `is_active` | BOOLEAN | 帳號狀態（預設 true） |

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    avatar_url VARCHAR(500),
    target_score INT DEFAULT 900,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true
);
```

---

### 2. `questions` - 題目資料表

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | BIGSERIAL | 主鍵 |
| `part` | SMALLINT | TOEIC Part（1-7） |
| `question_type` | VARCHAR(20) | 題型（listening/reading） |
| `category` | VARCHAR(50) | 題目分類（photos/短對話/單句 etc.） |
| `difficulty` | SMALLINT | 難度（1-5，1 最簡單） |
| `content` | TEXT | 題目內容（題目文字或音頻 URL） |
| `audio_url` | VARCHAR(500) | 聽力音頻 URL |
| `image_url` | VARCHAR(500) | 圖片 URL（Part 1, 2） |
| `options` | JSONB | 選項（JSON 陣列） |
| `correct_answer` | VARCHAR(10) | 正確答案（A/B/C/D） |
| `explanation` | TEXT | 詳解 |
| `tags` | JSONB | 標籤（如：商務、旅行、交通） |
| `source` | VARCHAR(100) | 來源（官方模擬題/練習題庫） |
| `usage_count` | INT | 使用次數（用於分析熱門題目） |
| `correct_rate` | DECIMAL(5,2) | 正確率（系統自動計算） |
| `created_at` | TIMESTAMP | 建立時間 |
| `updated_at` | TIMESTAMP | 更新時間 |

```sql
CREATE TABLE questions (
    id BIGSERIAL PRIMARY KEY,
    part SMALLINT NOT NULL CHECK (part BETWEEN 1 AND 7),
    question_type VARCHAR(20) NOT NULL CHECK (question_type IN ('listening', 'reading')),
    category VARCHAR(50),
    difficulty SMALLINT DEFAULT 3 CHECK (difficulty BETWEEN 1 AND 5),
    content TEXT NOT NULL,
    audio_url VARCHAR(500),
    image_url VARCHAR(500),
    options JSONB NOT NULL,
    correct_answer VARCHAR(10) NOT NULL,
    explanation TEXT,
    tags JSONB DEFAULT '[]',
    source VARCHAR(100),
    usage_count INT DEFAULT 0,
    correct_rate DECIMAL(5,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 索引
CREATE INDEX idx_questions_part ON questions(part);
CREATE INDEX idx_questions_difficulty ON questions(difficulty);
CREATE INDEX idx_questions_category ON questions(category);
CREATE INDEX idx_questions_type ON questions(question_type);
```

---

### 3. `user_progress` - 使用者進度追蹤

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | BIGSERIAL | 主鍵 |
| `user_id` | BIGINT | 關聯 users(id) |
| `part` | SMALLINT | TOEIC Part |
| `total_questions` | INT | 總題數 |
| `completed_questions` | INT | 完成題數 |
| `correct_count` | INT | 正確數 |
| `average_score` | DECIMAL(5,2) | 平均分數 |
| `mastery_level` | DECIMAL(5,2) | 掌握度（0-100%） |
| `time_spent_minutes` | INT | 總花費時間（分鐘） |
| `last_practiced_at` | TIMESTAMP | 最後練習時間 |
| `created_at` | TIMESTAMP | 建立時間 |
| `updated_at` | TIMESTAMP | 更新時間 |

```sql
CREATE TABLE user_progress (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    part SMALLINT NOT NULL CHECK (part BETWEEN 1 AND 7),
    total_questions INT DEFAULT 0,
    completed_questions INT DEFAULT 0,
    correct_count INT DEFAULT 0,
    average_score DECIMAL(5,2) DEFAULT 0,
    mastery_level DECIMAL(5,2) DEFAULT 0,
    time_spent_minutes INT DEFAULT 0,
    last_practiced_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, part)
);

CREATE INDEX idx_user_progress_user ON user_progress(user_id);
```

---

### 4. `answer_history` - 答題歷史

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | BIGSERIAL | 主鍵 |
| `user_id` | BIGINT | 關聯 users(id) |
| `question_id` | BIGINT | 關聯 questions(id) |
| `user_answer` | VARCHAR(10) | 使用者答案 |
| `is_correct` | BOOLEAN | 是否正確 |
| `time_spent_seconds` | INT | 答題耗時（秒） |
| `answered_at` | TIMESTAMP | 答題時間 |
| `session_id` | UUID | 練習場次 ID |

```sql
CREATE TABLE answer_history (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    question_id BIGINT NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    user_answer VARCHAR(10) NOT NULL,
    is_correct BOOLEAN NOT NULL,
    time_spent_seconds INT,
    answered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    session_id UUID
);

-- 索引（高效查詢）
CREATE INDEX idx_answer_history_user ON answer_history(user_id);
CREATE INDEX idx_answer_history_question ON answer_history(question_id);
CREATE INDEX idx_answer_history_date ON answer_history(answered_at);
CREATE INDEX idx_answer_history_session ON answer_history(session_id);

-- 用於分析使用者答題趨勢
CREATE INDEX idx_answer_history_user_date ON answer_history(user_id, answered_at DESC);
```

---

### 5. `user_statistics` - 每日統計

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | BIGSERIAL | 主鍵 |
| `user_id` | BIGINT | 關聯 users(id) |
| `stat_date` | DATE | 統計日期 |
| `total_questions` | INT | 當日總答題數 |
| `correct_count` | INT | 當日正確數 |
| `total_time_seconds` | INT | 當日總答題時間 |
| `listening_count` | INT | 聽力題數 |
| `reading_count` | INT | 閱讀題數 |
| `listening_correct` | INT | 聽力正確數 |
| `reading_correct` | INT | 閱讀正確數 |
| `sessions_count` | INT | 練習場次數 |
| `created_at` | TIMESTAMP | 建立時間 |
| `updated_at` | TIMESTAMP | 更新時間 |

```sql
CREATE TABLE user_statistics (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    stat_date DATE NOT NULL,
    total_questions INT DEFAULT 0,
    correct_count INT DEFAULT 0,
    total_time_seconds INT DEFAULT 0,
    listening_count INT DEFAULT 0,
    reading_count INT DEFAULT 0,
    listening_correct INT DEFAULT 0,
    reading_correct INT DEFAULT 0,
    sessions_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, stat_date)
);

CREATE INDEX idx_user_statistics_user_date ON user_statistics(user_id, stat_date DESC);
```

---

### 6. `spaced_repetition` - 艾賓浩斯遺忘曲線學習記錄

採用 SM-2 演算法（類似 Anki）追蹤複習時間。

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | BIGSERIAL | 主鍵 |
| `user_id` | BIGINT | 關聯 users(id) |
| `question_id` | BIGINT | 關聯 questions(id) |
| `ease_factor` | DECIMAL(4,3) | 難度係數（初始 2.5） |
| `interval` | INT | 複習間隔（天數） |
| `repetitions` | INT | 複習次數 |
| `next_review_date` | DATE | 下次複習日期 |
| `last_reviewed_at` | TIMESTAMP | 最後複習時間 |
| `status` | VARCHAR(20) | 狀態（new/learning/review/lapsed） |
| `lapses` | INT | 遺忘次數 |
| `created_at` | TIMESTAMP | 建立時間 |
| `updated_at` | TIMESTAMP | 更新時間 |

```sql
CREATE TABLE spaced_repetition (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    question_id BIGINT NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    ease_factor DECIMAL(4,3) DEFAULT 2.5,
    interval INT DEFAULT 0,
    repetitions INT DEFAULT 0,
    next_review_date DATE,
    last_reviewed_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'new' CHECK (status IN ('new', 'learning', 'review', 'lapsed')),
    lapses INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, question_id)
);

-- 查詢今日待複習題目
CREATE INDEX idx_spaced_repetition_review ON spaced_repetition(next_review_date) 
WHERE next_review_date <= CURRENT_DATE;

CREATE INDEX idx_spaced_repetition_user ON spaced_repetition(user_id);
```

---

### 7. `practice_sessions` - 練習場次

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | UUID | 主鍵（UUID） |
| `user_id` | BIGINT | 關聯 users(id) |
| `session_type` | VARCHAR(20) | 類型（practice/exam/topic） |
| `part` | SMALLINT | Part 範圍（如：1-7） |
| `question_count` | INT | 題目數量 |
| `correct_count` | INT | 正確數 |
| `total_time_seconds` | INT | 總時間 |
| `started_at` | TIMESTAMP | 開始時間 |
| `completed_at` | TIMESTAMP | 完成時間 |
| `status` | VARCHAR(20) | 狀態（in_progress/completed/abandoned） |

```sql
CREATE TABLE practice_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_type VARCHAR(20) DEFAULT 'practice',
    part_start SMALLINT,
    part_end SMALLINT,
    question_count INT DEFAULT 0,
    correct_count INT DEFAULT 0,
    total_time_seconds INT DEFAULT 0,
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'in_progress'
);

CREATE INDEX idx_practice_sessions_user ON practice_sessions(user_id);
CREATE INDEX idx_practice_sessions_date ON practice_sessions(started_at);
```

---

### 8. `question_collections` - 題目集合（可選題庫）

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | BIGSERIAL | 主鍵 |
| `name` | VARCHAR(100) | 集合名稱 |
| `description` | TEXT | 描述 |
| `part` | SMALLINT | 適用 Part |
| `question_ids` | JSONB | 題目 ID 陣列 |
| `is_public` | BOOLEAN | 是否公開 |
| `creator_id` | BIGINT | 建立者 |
| `created_at` | TIMESTAMP | 建立時間 |
| `updated_at` | TIMESTAMP | 更新時間 |

```sql
CREATE TABLE question_collections (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    part SMALLINT CHECK (part BETWEEN 1 AND 7),
    question_ids JSONB DEFAULT '[]',
    is_public BOOLEAN DEFAULT false,
    creator_id BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 資料表關係圖

```
┌─────────────────┐       ┌─────────────────────┐
│     users       │       │      questions      │
├─────────────────┤       ├─────────────────────┤
│ id (PK)         │◄──────│ id (PK)             │
│ email           │       │ part                │
│ username        │       │ question_type       │
│ password_hash   │       │ difficulty          │
│ ...             │       │ content             │
└────────┬────────┘       │ correct_answer      │
         │                │ ...                 │
         │                └──────────┬───────────┘
         │                           │
         │         ┌─────────────────┼─────────────────┐
         │         │                 │                 │
         ▼         ▼                 ▼                 ▼
┌─────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│  user_progress │ │   answer_history    │ │  spaced_repetition │
├─────────────────┤ ├─────────────────────┤ ├─────────────────────┤
│ id (PK)         │ │ id (PK)             │ │ id (PK)             │
│ user_id (FK)────┼─► user_id (FK)       │ │ user_id (FK)───────┼─►
│ part            │ │ question_id (FK)───┼─► question_id (FK)──┤
│ total_questions │ │ session_id (FK)   │ │ ease_factor         │
│ completed_      │ │ is_correct         │ │ interval            │
│   questions     │ │ time_spent_seconds│ │ next_review_date   │
│ mastery_level   │ │ answered_at        │ │ status              │
└─────────────────┘ └─────────────────────┘ └─────────────────────┘
         │
         │                ┌─────────────────────┐
         │                │  user_statistics   │
         └───────────────►├─────────────────────┤
                          │ id (PK)             │
                          │ user_id (FK)───────┘
                          │ stat_date           │
                          │ total_questions     │
                          │ correct_count       │
                          └─────────────────────┘

         │
         │                ┌─────────────────────┐
         └───────────────►│  practice_sessions  │
                          ├─────────────────────┤
                          │ id (PK, UUID)       │
                          │ user_id (FK)───────┘
                          │ session_type        │
                          │ started_at          │
                          │ completed_at        │
                          │ status              │
                          └─────────────────────┘
```

---

## ER 關係說明

| 關係 | 類型 | 說明 |
|------|------|------|
| users → user_progress | 1:N | 每個使用者每個 Part 只有一筆記錄 |
| users → answer_history | 1:N | 每個使用者可有多筆答題記錄 |
| users → spaced_repetition | 1:N | 每個使用者每題只有一筆記錄 |
| users → user_statistics | 1:N | 每個使用者每天一筆記錄 |
| users → practice_sessions | 1:N | 每個使用者可有多個練習場次 |
| questions → answer_history | 1:N | 每題可被多次回答 |
| questions → spaced_repetition | 1:N | 每題對應一筆複習記錄 |
| answer_history → practice_sessions | N:1 | 答案屬於某場練習 |

---

## 效能優化建議

### 1. 索引策略
- 常用查詢欄位建立索引（user_id, question_id, date）
- 部分索引：只索引需要複習的題目
- 複合索引：支援常用篩選條件

### 2. 分區表（Partitioning）
```sql
-- 對大表進行分區（超過百萬筆時考慮）
CREATE TABLE answer_history (
    ...
) PARTITION BY RANGE (answered_at);

-- 按月分區
CREATE TABLE answer_history_2026_01 PARTITION OF answer_history
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

### 3. 讀寫分離
- 主資料庫處理寫入
- 副本資料庫處理統計查詢、分析報表

### 4. 快取策略
- Redis 快取使用者進度、每日統計
- CDN 快取靜態資源（題目圖片、音頻）

---

## 未來擴展

| 功能 | 說明 |
|------|------|
| 多設備同步 | 透過 user_id 跨裝置同步進度 |
| 社群功能 | 加入 user_follows, comments 等表 |
| AI 推薦 | 擴展 questions 欄位，新增 AI 分析標籤 |
| 類神經網絡 | 預測使用者分數區間 |

---

## 總結

本設計採用 PostgreSQL，支援：
- ✅ 大量題目儲存與檢索
- ✅ 使用者進度完整追蹤
- ✅ 詳細答題歷史分析
- ✅ 艾賓浩斯遺忘曲線複習機制
- ✅ 高效能查詢優化
- ✅ 未來橫向擴展能力
