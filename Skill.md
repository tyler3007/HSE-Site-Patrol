---
name: "hse-site-patrol"
description: Analyze construction-site safety photos with AI grounded on Vietnamese and international OSH sources (law, decrees, circulars, QCVN/TCVN, ISO 45001, ILO, OSHA, company/project rules), let the user review and edit the findings, export the HSE Site Patrol Report Excel in the company template (3 sheets, photos embedded), and draft the email to contractors for HIGH-risk items. Use this skill whenever the user shares site photos or notes about unsafe conditions, mentions tuần tra / kiểm tra ATSKMT / báo cáo HSE / biên bản kiểm tra ATVSLĐ / Site Patrol Report / NCR / căn cứ pháp lý, asks to rate a violation (mức độ rủi ro, R=L×S), asks to export the patrol Excel, or asks to write the email/thư yêu cầu nhà thầu khắc phục (NTC, BMW, etc.), even if they only mention one of these steps.
---

# HSE Site Patrol

Turns site-patrol photos into three outputs the HSE supervisor sends to contractors. Reply to the user in Vietnamese unless they write in another language; keep report text in Vietnamese.

Workflow: **1. AI analysis of photos, grounded on legal sources -> 2. User reviews and edits -> 3. Excel report -> 4. Email draft for HIGH risks, which the user adjusts.** The user may ask for only one step; do just that step.

## 0. Context first

- Read the **project context** (files and notes the user added to the project: company HSE rules such as HSE-02, method statements, earlier reports). This is the first and most specific source. `references/hse-rules.md` is only a pointer to it. If no project rules are available, say so once and continue with legal sources.
- Ask once, in one message, for header data that is missing: project name, patrol date and time, inspectors, report number, contractor list. Never guess the contractor (Nhà thầu) of a finding.

## 1. Analyze the photos (AI + legal sources)

**A. Look.** For each photo describe only what is visible: the hazard, who is exposed, the consequence. No clear violation -> say so instead of inventing one. Several violations -> make the most severe the row and mention the others in Vấn đề.

**B. Ground each finding** in sources, in this priority:
1. **Project context** (company rules): tag `[Dự án]`.
2. **Vietnam**: Luật ATVSLĐ and its decrees/circulars, rules on safety in construction works, QCVN and TCVN for scaffolds, lifting equipment, electrical, fire prevention: tag `[Luật VN]` or `[QCVN/TCVN]`.
3. **International** as supplementary good practice: ISO 45001, ILO C167 and ILO construction code of practice, OSHA 29 CFR 1926, EN standards (scaffolds, edge protection): tag `[Quốc tế]`. Where Vietnamese law or the project rules are stricter, they win.

How to retrieve: use whatever retrieval tools this session has. A connected legal or OSH API/MCP connector the user set up comes first; otherwise web_search/web_fetch, preferring government portals and official gazettes, then legal databases. This skill holds no API keys. Search once per hazard topic (work at height, scaffold, electrical, lifting, fire, PPE...) and reuse it across photos.

Rules for citing:
- Cite only what you retrieved in this session. Never quote a document number, article or clause from memory; Vietnamese OSH rules change often (amendments, new penalty decrees, new lists of strict-requirement jobs).
- Check the status: in force, amended or replaced, and the effective date against the patrol date.
- Format: `<Document number> <short name>, Điều/Khoản/Mục <n>` plus the tag. Say whether it is mandatory (law, QCVN) or recommended (standard, company practice).
- Nothing found or no retrieval tool -> write "Chưa xác minh căn cứ" and leave the finding as an observation. Do not invent a citation.
- Do not label anything a "lỗi nặng / NCR nặng" unless the project context contains the list that defines it; otherwise say it may qualify and that the project's list decides.

**C. Fill one row per finding:**

| Field | Rule |
|---|---|
| Vị trí | Leave empty unless the user gives it. A photo rarely shows tower/floor reliably, and a wrong location sends a contractor to the wrong place. |
| Vấn đề | What is wrong and the consequence, one or two sentences. |
| Giải pháp | Concrete actions, one per line starting with "- ", following the hierarchy of controls named in the template heading (loại bỏ, thay thế, kiểm soát kỹ thuật, kiểm soát hành chính, PPE); include only the levels that apply. Prefer the control the sources require over generic advice. |
| Loại rủi ro | Exactly one value from the template's `Loại rủi ro` sheet: Ngã cao, Vật rơi, Đổ sập, Nâng hạ, Va chạm, Điện giật, Cháy nổ, Hành vi, PPE, Sạt lở, Văng bắn, Vấp ngã, Ứng phó khẩn cấp, Môi trường, Vệ sinh, Tổ chức mặt bằng. The script rejects any other spelling. |
| L, S, R | L = 1 Rất hiếm, 2 Hiếm, 3 Có thể, 4 Khả năng cao, 5 Chắc chắn. S = 1 Không đáng kể, 2 Nhẹ, 3 Trung bình, 4 Nghiêm trọng, 5 Thảm khốc. R = L x S. Cao R >= 15, Vừa 8-12, Thấp <= 6 (template legend). Raise L when workers are exposed right now; raise S for large fall/drop heights or people below. If the project context has its own calibration, follow it. |
| Thời hạn | Cao: 1-3 days after the patrol date; Vừa: 4-7; Thấp: 7+. Format dd/mm/yyyy. Project rules override. |
| Nhà thầu | Goes in "Thực hiện bởi", e.g. NTC-MEP. Ask if unclear. |
| Căn cứ | Citation(s) from step B. Not a template column: keep it in the chat table and in `input.json` (`basis`), not in the Excel. |

## 2. User review

Show the findings as a table in chat: Stt | Nhà thầu | Vị trí | Vấn đề | Giải pháp (shortened) | Loại | L x S = R (mức) | Hạn | Căn cứ. Ask for corrections and wait. The user edits quickly and expects to. Apply changes, re-show only the changed rows, and export only after the user confirms. Keep the confirmed rows in `input.json`.

## 3. Excel report

Always generate the file from `assets/template.xlsx` with `scripts/build_report.js`. The template carries the header, legend, merged cells, footer, print titles and the `Loại rủi ro` / `Mức độ rủi ro` sheets, and must match the company form. Do not rebuild the workbook with openpyxl or from scratch (it drops the dropdown extension and drawings).

```bash
cd /home/claude && npm install exceljs@4.4.0   # once per session
node <skill>/scripts/build_report.js input.json "261120 MTT.HSE.SP.044 Site Patrol Report - NTC-MEP.xlsx" [--contractor NTC-MEP]
```

`input.json` (see `assets/example-input.json`, which reproduces the template's sample row; `image` paths resolve relative to the JSON file):

```json
{"project":"TỨ GIÁC BẾN THÀNH / The One Central Saigon","time":"8:30 - 11:00","date":"20/11/2026","who":"MAG, ART, NTC, BMW",
 "items":[{"con":"NTC-MEP","pos":"","problem":"...","solution":"- ...\n- ...","cat":"Ngã cao","L":4,"S":4,"due":"22/11/2026","image":"photo.jpg","basis":"..."}]}
```

What the script does: removes the template's sample row and photo; writes one flat numbered list (no contractor heading rows); computes R = L x S (or takes `R`); colors the R cell red / yellow / green; puts the violation photo in the "Vi phạm" column and leaves the corrective-action column empty for the contractor; fills "Thực hiện bởi" and the deadline; sizes each row to its text and photo; validates `cat` against the `Loại rủi ro` sheet and adds the dropdown; updates the risk counts on that sheet. `basis` is ignored. `--contractor` keeps only that contractor's items.

File name: `yymmdd MTT.HSE.SP.<number> Site Patrol Report - <contractor>.xlsx`. The usual practice is one workbook per contractor; ask if the user wants one combined file instead. The report number increments each patrol; ask if unknown.

After writing, render a page to an image (soffice -> pdf -> pdftoppm) and look at it once, then use present_files.

## 4. Email to contractors (draft for the user to adjust)

Only HIGH-risk items (R >= 15) go in the email; Vừa/Thấp stay in the Excel. Group by contractor in this format:

```
Các vấn đề dưới đây có mức độ rủi ro CAO, đề nghị các nhà thầu khắc phục khẩn cấp, đúng hạn:
Nhà thầu NTC:
1. <Vị trí> – <Vấn đề, with the consequence>.
→ Yêu cầu: <actions in one sentence>. Hạn: d/m/yyyy.
2. ...
Nhà thầu BMW:
1. ...
```

Keep the position in the email even if the Excel column is blank, because the contractor needs to know where to go; ask once for the ones missing. Deadlines must equal the Excel deadlines. Subject: `[HSE] Các vấn đề rủi ro CAO cần khắc phục khẩn cấp - Biên bản tuần tra dd/mm/yyyy (<report number>)`. Add a closing request for corrective photos before the deadline.

This is a **draft**: show it with the draft-message tool when available (otherwise in chat), then apply the user's edits and show it again until they are happy. Never send it. Create a mail-service draft only if the user asks.

## Reference

`references/hse-rules.md` points to the rules in the project context, which the user maintains. Read the project context before analyzing; do not treat that file as the rules themselves.
