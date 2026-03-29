核心面向

  重構

  1. 方法 / 類別重構 — 消除 God Class、提取 Service、命名規範（C# 慣例）
  2. 依賴注入重構 — 正確使用 DI lifetime（Singleton / Scoped / Transient）、避免 Service Locator

  架構

  3. 分層架構 — Controller → Service → Repository，各層職責與邊界
  4. Clean Architecture — Use Case / Domain / Infrastructure 在 .NET 專案結構的實踐
  5. CQRS — MediatR 的使用方式、Command / Query 分離

  .NET 特定

  6. EF Core 最佳實踐 — 查詢最佳化、N+1、Migration 管理、DbContext lifetime
  7. Middleware / Filter / Attribute — 橫切關注點的正確位置
  8. 錯誤處理 — Global Exception Handler、Problem Details（RFC 7807）、Result Pattern

  品質

  9. 測試策略 — xUnit、Moq、Integration Test with WebApplicationFactory
  10. API 設計 — Minimal API vs Controller、版本管理、Swagger 整合
