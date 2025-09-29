# 設計文件 - 範例功能

## 概述

本文件描述用戶註冊和登入功能的技術設計方案。

## 架構

系統採用三層架構：
- **展示層** - 用戶介面和 API 端點
- **業務邏輯層** - 用戶管理服務
- **資料存取層** - 資料庫操作

## 元件和介面

### 用戶管理服務
```typescript
interface UserService {
  register(email: string, password: string): Promise<User>
  login(email: string, password: string): Promise<AuthToken>
  validatePassword(password: string): boolean
}
```

### 用戶模型
```typescript
interface User {
  id: string
  email: string
  passwordHash: string
  isEmailVerified: boolean
  createdAt: Date
  lastLoginAt?: Date
}
```

## 資料模型

### 用戶表
- `id` (UUID, Primary Key)
- `email` (String, Unique)
- `password_hash` (String)
- `is_email_verified` (Boolean)
- `created_at` (Timestamp)
- `last_login_at` (Timestamp, Nullable)

### 登入嘗試表
- `id` (UUID, Primary Key)
- `user_id` (UUID, Foreign Key)
- `ip_address` (String)
- `success` (Boolean)
- `attempted_at` (Timestamp)

## 錯誤處理

- **無效電子郵件格式** - 回傳 400 錯誤和詳細訊息
- **密碼不符合要求** - 回傳 400 錯誤和密碼要求說明
- **電子郵件已存在** - 回傳 409 錯誤
- **登入憑證錯誤** - 回傳 401 錯誤
- **帳號被鎖定** - 回傳 423 錯誤

## 測試策略

### 單元測試
- 用戶服務的所有方法
- 密碼驗證邏輯
- 電子郵件格式驗證

### 整合測試
- API 端點的完整流程
- 資料庫操作
- 電子郵件發送功能

### 端對端測試
- 完整的註冊流程
- 完整的登入流程
- 錯誤情境處理