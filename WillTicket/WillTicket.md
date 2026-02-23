# WillTicket

WillTicket 是一個提供網路售票服務的平台，協助活動主辦方 (企業戶) 在線上販售活動票券，並提供消費者流暢的搶票與購票體驗。消費者透過 Google OAuth2 登入，同時也提供額外帳密登入。

該平台提供兩個 Web Base 的介面, 分別是 ticket & tadmin, 供一般消費者與管理員/企業戶使用, 其使用者包含：

|角色|說明|主要操作|登入方式|
| ---|---|---|---|
|消費者（Consumer）|一般使用者|瀏覽活動、搶票、付款、查看訂單|Google OAuth2|
|企業戶（Enterprise）|活動主辦方|申請活動、查看銷售數據| ENTERPRISE Role|
|管理員（Admin）|平台管理者|審核活動、管理平台|Google OAuth2 + ADMIN Role|


tadmin 為後台管理系統, 提供的服務包含:
* 管理員與企業戶登入
* 後台 RBAC 與功能管理
* 活動申請與審核
* 付款整合
* 訂單管理

