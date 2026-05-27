

# API payload

```
type ExportRequest = {
  startAt: string; // UTC ISO string
  endAt: string;   // UTC ISO string, exclusive

  timezoneOffsetMinutes: number;
  timezoneOffsetLabel: string;
};
```

## 範例：

```
{
  "startAt": "2025-11-30T16:00:00.000Z",
  "endAt": "2025-12-30T16:00:00.000Z",
  "timezoneOffsetMinutes": 510,
  "timezoneOffsetLabel": "UTC+8.5"
}
```

## 後端用途分開：

`timezoneOffsetMinutes`：用來轉換 export 檔案內的時間
`timezoneOffsetLabel`：用來顯示在 export title / 檔名 / header

例如 export title：

`Export Date Range: May 21, 2026 00:00:00 -  May 25, 2026 00:00:00 (UTC+8.5)`

後端不用自己組 title，也不用擔心 +8.5 parsing 問題。

---