# ARCHITECTURE DOCUMENTATION - Product Search & List

## Tổng Quan

Dự án này gồm 2 trang HTML chính:

1. **product-search.html** (1090 dòng)
   - Tìm kiếm sản phẩm từ API
   - Cài đặt hiển thị grid
   - Preview danh sách sản phẩm

2. **product-list.html** (540 dòng)
   - Hiển thị grid sản phẩm
   - Pagination
   - Responsive design

---

## Quick Start - Hiểu Cấu Trúc

### 1. Settings Flow (2 phút)

Settings được lưu vào **localStorage** với key `'displaySettings'`:

```javascript
{
    columns: 4,          // Số cột (1-8)
    rows: 2,             // Số hàng (1-6)
    gap: 15,             // Khoảng cách (5-50px)
    itemsPerPage: 8      // Tính tự động = columns × rows
}
```

**Flow**:
- User thay đổi settings trong product-search.html
- Click "Lưu Cài Đặt" → `applySettings()` saves to localStorage
- product-list.html detects change → re-renders grid
- 3 cách sync: storage event, setInterval (5s), manual reload

### 2. CSS Grid Layout (2 phút)

```css
.product-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);  /* 4 cột bằng nhau */
    grid-template-rows: repeat(2, 1fr);     /* 2 hàng bằng nhau */
    gap: 15px;                               /* Khoảng cách */
    height: calc(100vh - 30px);              /* Fill viewport */
}
```

Mỗi grid-item:
```
┌─────────────────────────┐
│ image (aspect-ratio: 1) │
├─────────────────────────┤
│ name (min-height: 36px) │
├─────────────────────────┤
│ stats (3 columns)       │  ← margin-top: auto (push down)
└─────────────────────────┘
```

### 3. Key Functions (5 phút)

**product-list.html**:
- `loadSettings()` - Tải settings từ localStorage
- `applySettings()` - Áp dụng grid styles
- `updateProductGrid()` - Render items với pagination

**product-search.html**:
- `applySettings()` - Save settings vào localStorage
- `loadProductDetails()` - Fetch từ API
- `addProductToList()` - Thêm vào savedProducts

---

## File Locations (Quick Reference)

| Mục đích | File | Lines |
|---------|------|-------|
| Settings object | product-list.html | 361-365 |
| Load/Apply settings (display) | product-list.html | 368-383 |
| Load/Apply settings (save) | product-search.html | 1014-1038 |
| Settings UI HTML | product-search.html | 506-534 |
| Grid CSS | product-list.html | 34-42 |
| Grid Item CSS | product-list.html | 44-126 |
| Sync mechanism | product-list.html | 501-536 |

---

## Thêm Tính Năng Điều Chỉnh Chiều Cao (6 bước)

### Bước 1: Thêm itemHeight vào settings
```javascript
// product-list.html line ~361
let displaySettings = {
    columns: 4,
    rows: 2,
    gap: 15,
    itemHeight: 500  // ← ADD THIS
};
```

### Bước 2: Thêm UI input
```html
<!-- product-search.html, add before Apply Button -->
<div class="setting-item">
    <label class="setting-label">📏 Chiều Cao Item (px)</label>
    <input type="number" class="setting-input" id="settingItemHeight" 
           value="500" min="200" max="1000">
</div>
```

### Bước 3: Update applySettings (product-search.html)
```javascript
function applySettings() {
    const itemHeight = parseInt(document.getElementById('settingItemHeight').value) || 500;
    
    const settings = {
        columns: ...,
        rows: ...,
        gap: ...,
        itemHeight: itemHeight,  // ← ADD THIS
        itemsPerPage: columns * rows
    };
    
    localStorage.setItem('displaySettings', JSON.stringify(settings));
}
```

### Bước 4: Update loadSettings (product-search.html)
```javascript
function loadSettings() {
    const parsed = JSON.parse(settings);
    document.getElementById('settingItemHeight').value = parsed.itemHeight || 500;  // ← ADD THIS
}
```

### Bước 5: Update applySettings (product-list.html)
```javascript
function applySettings() {
    const productGrid = document.getElementById('productGrid');
    productGrid.style.gridTemplateColumns = `repeat(${displaySettings.columns}, 1fr)`;
    productGrid.style.gridTemplateRows = `repeat(${displaySettings.rows}, 1fr)`;
    productGrid.style.gap = `${displaySettings.gap}px`;
    productGrid.style.gridAutoRows = `${displaySettings.itemHeight}px`;  // ← ADD THIS
}
```

### Bước 6: Test
- Thay đổi itemHeight trong product-search.html
- Click "Lưu Cài Đặt"
- Mở product-list.html
- Grid items phải có chiều cao mới

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   BROWSER                                   │
│                                                              │
│  ┌──────────────────────┐  ┌──────────────────────────────┐ │
│  │ product-search.html  │  │ product-list.html             │ │
│  │                      │  │                               │ │
│  │ Settings UI          │  │ Grid Display                  │ │
│  │ Search                │  │ Pagination                   │ │
│  │ Product Details      │  │ Sync Listener                │ │
│  └──────────────────────┘  └──────────────────────────────┘ │
│          ↓                            ↑                      │
│     applySettings()          loadSettings()                 │
│          ↓                            ↑                      │
│  ┌─────────────────────────────────────────────────────────┐│
│  │         localStorage                                    ││
│  │  displaySettings {                                      ││
│  │    columns, rows, gap, itemsPerPage                    ││
│  │  }                                                      ││
│  │  savedProducts [...]                                   ││
│  └─────────────────────────────────────────────────────────┘│
│                            ↑                                 │
│                      API Calls                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
                  ┌──────────────────┐
                  │  API Server      │
                  │ (tomato.tpos.vn) │
                  └──────────────────┘
```

---

## Important Notes

### 1. Inline Styles Override Media Queries
JS settings sử dụng `element.style.*` có độ ưu tiên cao hơn CSS rules:
```javascript
// Cái này sẽ OVERRIDE media queries
productGrid.style.gridTemplateColumns = `repeat(${displaySettings.columns}, 1fr)`;
```

### 2. Storage Event Only Fires in Other Tabs
```javascript
// Fired khi localStorage thay đổi từ OTHER tabs/windows
window.addEventListener('storage', (e) => {
    if (e.key === 'displaySettings') {
        // React to changes
    }
});
```

### 3. Grid Item Height Calculation
```
Total height = calc(100vh - 30px)
Available = Total - (gap × (rows - 1))
Item height = Available / rows
```

Example (viewport 1000px, 2 rows, 15px gap):
```
Total = 1000 - 30 = 970px
Available = 970 - 15 = 955px
Item height = 955 / 2 = 477.5px
```

### 4. Pagination Uses itemsPerPage
```javascript
itemsPerPage = columns × rows  // Not related to itemHeight!
```

---

## Testing Checklist

Sau khi implement bất cứ thay đổi gì:

- [ ] Settings UI loads đúng giá trị từ localStorage
- [ ] Khi save settings, localStorage được update
- [ ] Grid re-renders với settings mới
- [ ] Pagination hoạt động đúng (itemsPerPage = columns × rows)
- [ ] Responsive: hoạt động trên mobile/tablet/desktop
- [ ] Cross-tab sync: mở 2 tab cùng lúc, thay đổi settings
- [ ] Empty state: hiển thị khi không có sản phẩm
- [ ] Performance: không lag khi render 100+ items

---

## Common Issues & Solutions

### Issue: Settings không được lưu
- Check: applySettings() function có gọi `localStorage.setItem()` không?
- Check: key name là 'displaySettings' không?

### Issue: Grid không update khi thay đổi settings
- Check: loadSettings() được gọi không?
- Check: applySettings() được gọi không?
- Check: có storage event listener không?

### Issue: Pagination sai
- Check: itemsPerPage = columns × rows?
- Check: currentPage reset về 1 không?

### Issue: Responsive không hoạt động
- Check: JS settings override media queries không?
- Check: window.innerWidth được tính đúng không?

---

## Future Improvements

1. **Add itemHeight setting** (Recommended)
   - Allow users to customize grid item height
   - Use `gridAutoRows` instead of `gridTemplateRows`

2. **Add image aspect ratio setting**
   - Allow 1:1, 16:9, 4:3, 9:16 ratios
   - Dynamic CSS application

3. **Add text size presets**
   - Small, Medium, Large
   - Store as class name

4. **Save user preferences**
   - Remember last settings on first visit
   - Default values based on screen size

5. **Add keyboard shortcuts**
   - Arrow keys: navigate pages
   - +/-: adjust columns/rows
   - Spacebar: toggle fullscreen

---

## Files & Documentation

- **Code Structure Analysis**: See `/tmp/code_structure_analysis.md`
- **Visual Flow Diagrams**: See `/tmp/visual_flow.md`
- **Implementation Guide**: See `/tmp/implementation_guide.md`
- **File Index & Reference**: See `/tmp/FILE_INDEX.md`

---

## Contact & Support

For questions about:
- **Settings system**: Check product-search.html lines 1014-1038
- **Grid layout**: Check product-list.html lines 34-126
- **Sync mechanism**: Check product-list.html lines 501-536
- **Responsive design**: Check media queries (lines 237-346)

---

Generated: 2025-11-08
Status: Current (as of commit 005b584)
