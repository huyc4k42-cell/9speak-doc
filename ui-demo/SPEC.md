# Spec: Responsive Assistant Panel (phong cách NotebookLM)

Demo tham khảo: `ui-demo/index.html` (mở trực tiếp bằng trình duyệt, kéo giãn cửa sổ để test).
Đây là prototype tĩnh (HTML/CSS/JS thuần) để chứng minh hành vi — **không phải code để merge**, dev port lại logic vào React theo cấu trúc component thật (`PracticeQuestionShell.tsx`, `PracticeAssistantRail.tsx`, `PracticeEvaluationPanel.tsx`).

## 1. Nguyên tắc cốt lõi

1. Panel phải là **một feed duy nhất, theo thứ tự thao tác** (bấm sau → nằm dưới), không phải nhiều state rời rạc.
2. Bậc hiển thị (breakpoint) dựa trên **bề rộng container của app-shell**, không phải `window.innerWidth` — dùng CSS `@container`, để panel co đúng khi sidebar app thu/mở chứ không chỉ khi resize cửa sổ.
3. Panel có 2 cách thu gọn độc lập:
   - **Tự động theo bậc** (Compact → tab; Medium → rail-icon + overlay).
   - **Thủ công** (Wide vẫn có thể bấm chevron để user tự thu gọn/mở, giống NotebookLM).

## 2. Bảng breakpoint

| Bậc | Điều kiện (container width) | Cấu trúc `.app-body` | Hành vi panel |
|---|---|---|---|
| Compact | `< 768px` | `flex-direction: column` | Tab pill "Luyện nói / AI hỗ trợ" full-width, đổi bằng `display:none` theo `data-active-tab` trên `.app-body` |
| Medium | `768px – 1199px` | `flex-direction: row` | Panel `position: fixed`, mặc định width = `--rail-width` (64px, chỉ icon dọc). Bấm icon bất kỳ → `width` giãn thành `min(420px, 100%-32px)`, có backdrop mờ đè lên conversation. Đóng bằng backdrop hoặc chevron. |
| Wide | `>= 1200px` | `flex-direction: row` | Panel là cột thật trong layout (`width: 380px` mặc định). Có thể bấm chevron để tự thu về `--rail-width` (rail-icon), bấm lại panel để mở ra. Không có backdrop (không che conversation). |

Container query gốc: bọc toàn bộ shell trong 1 phần tử `container-type: inline-size; container-name: app-shell;`, mọi `@media` đổi thành `@container app-shell (...)`.

## 3. Cấu trúc DOM / component (map sang React)

```
AppShell                              → PracticeQuestionShell (thêm container-type)
 ├─ Header
 ├─ AppBody (data-active-tab)         → giữ nguyên, chỉ đổi flex-direction theo bậc
 │   ├─ MobileTabs                    → PracticeDetailMobileTabs (đã có sẵn, KHÔNG cần đổi)
 │   ├─ ConversationCol               → cột trái hiện tại (PracticeEvaluationPanel + record bar)
 │   ├─ Backdrop                      → MỚI — chỉ render/active ở bậc Medium
 │   └─ AssistantPanel                → PracticeAssistantRail
 │       ├─ Head (title + collapseBtn + railIcons)   → railIcons là MỚI
 │       ├─ Feed (danh sách card)                     → thay cho `body` + `analysisCards` rời rạc hiện tại
 │       └─ Actions (Câu mẫu / Từ vựng / Ghi chú)      → giữ nguyên AssistantSectionButton
```

### State model đề xuất (thay cho 3 nguồn rời rạc hiện tại trong `PracticeAssistantRail.tsx`)

```ts
type PanelFeedItem =
  | { id: "action:sample"; kind: "sample" }
  | { id: "action:vocab"; kind: "vocab" }
  | { id: "action:note"; kind: "note" }
  | { id: `criterion:${string}`; kind: "analysis"; criterionKey: keyof IELTSCriteriaScores; evaluation: PracticeEvaluationResult }
  | { id: `rewrite:${string}`; kind: "rewrite"; requestId: string };

type PanelLayoutMode = "tabs" | "rail" | "split";
```

- `items: PanelFeedItem[]` — 1 mảng duy nhất, append theo thứ tự bấm. Thay thế `body: RailBlock[]` + prop `analysisCards` + 2 `useEffect` đồng bộ hiện có ở [PracticeAssistantRail.tsx:312](../9speak-web/features/speaking/components/practice/PracticeAssistantRail.tsx) và dòng 335–358.
- `mode: PanelLayoutMode` — derive từ `ResizeObserver` trên app-shell (demo dùng ngưỡng 768/1200, xem hàm `ResizeObserver` cuối file `index.html`), KHÔNG hardcode qua `window.matchMedia`.
- `expanded: boolean` (chỉ có ý nghĩa khi `mode === "rail"`), `collapsed: boolean` (chỉ có ý nghĩa khi `mode === "split"`).

## 4. Interaction spec

| Hành động | Compact | Medium | Wide |
|---|---|---|---|
| Bấm nút "Câu mẫu"/"Từ vựng"/"Ghi chú" | Thêm card vào feed, feed tự cuộn xuống cuối | Giống Compact + panel tự bung từ rail → drawer nếu đang thu gọn | Thêm card vào feed, tự cuộn |
| Bấm điểm tiêu chí (cột trái) | Chuyển tab sang "AI hỗ trợ" rồi thêm card | Panel tự bung drawer + thêm card | Thêm card vào feed |
| Bấm lại cùng 1 tiêu chí điểm | Đóng card đó | Đóng card đó | Đóng card đó |
| Bấm nút "X" trên card | Xoá card khỏi feed | Xoá card khỏi feed | Xoá card khỏi feed |
| Bấm chevron panel | Ẩn (n/a — dùng tab) | Đóng drawer (về rail) | Toggle collapse (rail icon ⇄ full panel) |
| Đổi câu hỏi (`storageKey` đổi) | Reset toàn bộ `items = []`, về tab "Luyện nói" | Reset `items`, đóng drawer | Reset `items`, giữ nguyên trạng thái collapsed/expanded của panel (không phải reset UI panel, chỉ reset nội dung feed) |

## 5. Animation

- Card mới: `opacity 0→1, translateY(8px→0)` trong `260ms cubic-bezier(0.16,1,0.3,1)` (biến `--transition-panel`).
- Card mới có viền highlight (`box-shadow` xanh) trong ~1.4s rồi tự tắt — giúp user nhận ra vừa có nội dung mới mà không cần đọc kỹ.
- Card bị đóng: fade + dịch nhẹ lên trên trong 140ms rồi mới remove khỏi DOM (tránh "giật" do unmount ngay lập tức).
- Panel đổi width (rail ⇄ full): `transition: width 260ms` — do dùng `width` trực tiếp (không phải `flex-basis`) nên animate mượt trên mọi trình duyệt.
- Feed tự `scrollTo({ behavior: "smooth" })` xuống cuối mỗi khi thêm item.

## 6. Vấn đề cần lưu ý khi implement thật (đã kiểm chứng qua bug thực tế lúc build demo)

1. **Event bubbling khi nút nằm trong vùng có click-to-expand**: nút "collapse" nằm bên trong `assistant-panel`, và panel có listener "click vào rail để mở ra". Nếu không `stopPropagation()` trên nút collapse, click sẽ bị bắt 2 lần (đóng rồi tự mở lại ngay). Đây là bug thật đã gặp khi build demo — khi port sang React, để ý thứ tự event handler tương tự (`onClick` trên nút con vs. `onClick` trên panel cha).
2. **`display: flex` mặc định là row**: nếu `.app-body` (hoặc component tương đương) không set `flex-direction: column` ở bậc Compact, các phần tử ẩn/hiện bằng `display:none` vẫn là flex-sibling theo hàng ngang, khiến phần tử còn lại bị `align-items: stretch` kéo giãn sai hình dạng (bug thật đã gặp: tab pill bị kéo cao hết màn hình). Luôn kiểm tra `flex-direction` khi đổi bậc responsive có đổi cả hướng layout.
3. **Container query cần `container-type: inline-size` trên đúng phần tử bọc ngoài cùng** — nếu đặt nhầm vào phần tử con, `@container` sẽ không nhận diện được ngưỡng.
4. **Reset feed theo câu hỏi**: phải reset `items` NHƯNG không reset `collapsed`/`expanded` của panel (người dùng đang để rail thu gọn thì chuyển câu vẫn nên giữ nguyên trạng thái thu gọn đó).
5. **Pending/skeleton state** (vocab/grammar corrections đang chờ dữ liệu async) khi panel đang ở dạng rail-icon (Medium/Wide-collapsed) — quyết định rõ: vẫn chạy ngầm và hiện badge "có cập nhật" trên icon, hay tạm dừng hiển thị. Demo hiện dùng dấu chấm xanh trên icon để báo "đã có nội dung"; nên tái dùng dấu hiệu này cho trạng thái "đang loading" (ví dụ chấm nhấp nháy) để không mất thông tin khi thu gọn.
6. **`localStorage` note không được mất nội dung** khi panel chuyển giữa rail/drawer — không unmount component chứa `<textarea>`, chỉ ẩn bằng CSS (`display:none` qua attribute selector), đúng như demo đang làm (feed luôn nằm trong DOM, CSS chỉ ẩn phần bao ngoài).
7. **Test riêng cả Part 1/2/3**: mỗi part truyền props hơi khác cho shell (`promptInsideScroll`, `promptHeaderFlush`), đổi cấu trúc layout ảnh hưởng cả 3 — cần QA từng part sau khi port.

## 7. Việc KHÔNG nằm trong demo này (cần làm rõ thêm trước khi code thật)

- Animation/logic thật cho "rewrite card" (Cải thiện/Mở rộng câu trả lời) — demo chỉ mock tĩnh, code thật phải xử lý trạng thái pending → success/failed bất đồng bộ.
- Đồng bộ giữa `criterionAnalysisCards` (hiện quản lý ở từng trang Part1/2/3) và feed hợp nhất — cần quyết định nơi sở hữu state duy nhất (khuyến nghị: hook dùng chung `usePracticeAssistantFeed`, xem phần 3).
- Accessibility: focus trap khi drawer mở (Medium), `aria-live="polite"` khi feed có card mới, đảm bảo `Esc` đóng drawer/overlay.
