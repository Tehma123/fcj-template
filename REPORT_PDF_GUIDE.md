# Hướng dẫn tự động tạo báo cáo PDF (VI/EN)

Repo này đã được bổ sung tính năng **tự động sinh báo cáo PDF** (song ngữ Việt/Anh) trực tiếp từ nội dung Hugo trong `content/`, để nộp cho trường (mẫu HCMUT, môn CO3335). Tính năng và các file dưới đây được lấy (copy) từ `fcaj-hcmut-template/` — chỉ phần **logic tạo report**, không copy nội dung cá nhân (không đụng tới `content/` của repo này).

## 1. Cơ chế hoạt động

```
content/ (.md)  ──►  scripts/convert_hugo_to_latex.py  ──►  report/generated/{en,vi}/*.tex
                                                                        │
                                                                        ▼
                                                      report/main.tex, main_en.tex
                                                                        │
                                                                        ▼ (latexmk)
                                                        report_vn.pdf, report_en.pdf
```

1. `scripts/convert_hugo_to_latex.py` tự động quét toàn bộ `content/`, tìm mọi trang `_index.md` / `_index.vi.md`, bóc tách frontmatter, chuyển shortcode Hugo (`{{% notice warning %}}` → khung LaTeX màu vàng) và dùng **pandoc** (kèm `scripts/hugo-notice.lua`) để convert Markdown → LaTeX.
2. Script sinh ra file tổng hợp `report/generated/content_body_{en,vi}.tex`, gom các trang theo từng section (`1-Worklog`, `2-Proposal`, ...) với heading tương ứng.
3. `report/main.tex` (bản tiếng Việt) và `report/main_en.tex` (bản tiếng Anh) là 2 file LaTeX khung sẵn (trang bìa HCMUT, mục lục, header/footer...), `\input` file tổng hợp ở bước 2, cộng với:
   - `report/ChuongTrinhThucTap(.en).tex` — chèn PDF biểu mẫu D2 nếu có.
   - `report/KetQuaXetTuyen(.en).tex` — chèn PDF biểu mẫu D3 nếu có.
   - `report/TongKet(.en).tex` — khối chữ ký cuối báo cáo.

Script **không cần sửa gì thêm khi bạn thêm trang mới** — nó tự dò `content/` mỗi lần chạy.

## 2. Chạy sinh PDF ở máy local

Cài đặt (Ubuntu/Debian):

```bash
sudo apt-get install -y pandoc texlive-latex-recommended texlive-latex-extra \
  texlive-fonts-recommended texlive-lang-other latexmk
pip install pyyaml
```

macOS:

```bash
brew install pandoc
brew install --cask mactex
pip install pyyaml
```

Chạy:

```bash
# 1. Convert Hugo content -> LaTeX
python3 scripts/convert_hugo_to_latex.py

# 2. Build PDF tiếng Việt (chạy 3 lần để mục lục/tham chiếu đúng)
cd report
latexmk -pdf -interaction=nonstopmode main.tex
latexmk -pdf -interaction=nonstopmode main.tex
latexmk -pdf -interaction=nonstopmode main.tex
cp main.pdf ../report_vn.pdf

# 3. Build PDF tiếng Anh
latexmk -pdf -interaction=nonstopmode main_en.tex
latexmk -pdf -interaction=nonstopmode main_en.tex
latexmk -pdf -interaction=nonstopmode main_en.tex
cp main_en.pdf ../report_en.pdf
```

Kết quả: `report_vn.pdf` và `report_en.pdf` ở thư mục gốc repo.

## 3. Tự động chạy trên GitHub Actions

Job **`build-pdf`** đã được thêm vào `.github/workflows/hugo.yml` (chạy song song với job `build-deploy` deploy site có sẵn), mỗi khi push lên `main`:

1. Cài TeX Live + pandoc trên runner Ubuntu.
2. Chạy `convert_hugo_to_latex.py`, build 2 bản PDF (VI/EN) như ở mục 2.
3. Đóng gói mã nguồn LaTeX đã sinh vào `latex_source.zip`.
4. Tạo/cập nhật **GitHub Release** tag `latest`, đính kèm `report_vn.pdf`, `report_en.pdf`, `latex_source.zip`.

→ Sau mỗi lần push, vào tab **Releases** của repo trên GitHub là tải được PDF mới nhất, không cần build tay.

## 4. Điều khiển nội dung xuất ra PDF bằng frontmatter

Có thể thêm các field sau vào frontmatter của từng `_index.md` để tùy chỉnh riêng cho bản PDF (không ảnh hưởng trang web Hugo):

**Loại bỏ hẳn 1 trang khỏi PDF** (và mọi trang con nếu là section cha):

```yaml
includeInReport: false
```

**Chỉ giữ một số cột của bảng markdown** (bảng dài dễ vỡ layout PDF):

```yaml
reportTableColumns:
  - Day
  - Task
  - Completion Date
```

**Chỉ giữ một số heading trên trang** (ẩn bớt phần không cần trong PDF):

```yaml
reportHeadings:
  - Week 1 Objectives
  - Week 1 Achievements
```

**Bảng worklog dùng renderer LaTeX riêng** (đẹp hơn bảng mặc định của pandoc, tự bọc `<br>`/`-`/`+` thành gạch đầu dòng):

```yaml
reportType: worklog
reportTableColumns:
  - Day
  - Task
  - Completion Date
```

(Các field trên đều có alias tương đương, ví dụ `include_in_report`, `report_table_columns`, `report_headings`, `report_type` — xem chi tiết trong `scripts/convert_hugo_to_latex.py`.)

## 5. Việc cần tự bổ sung

- **Biểu mẫu D2/D3 của trường**: đặt file `formD2.pdf` và `formD3.pdf` vào `report/form/` (thư mục này chưa có sẵn, cần tự tạo). Nếu thiếu, LaTeX sẽ tự chèn 1 trang placeholder "Đính kèm biểu mẫu D2/D3 tại đây" thay vì lỗi.
- **Trang bìa** (`report/main.tex`, `report/main_en.tex`): các dòng còn để chấm "...." (ngành, chương trình đào tạo, đơn vị thực tập, tên sinh viên, MSSV...) cần tự điền tay — đây là thông tin cá nhân nên không được tự động điền từ nội dung của repo mẫu.

## 6. Danh sách file đã thêm vào repo

```
scripts/
├── convert_hugo_to_latex.py   # script chính: Hugo content -> LaTeX
└── hugo-notice.lua            # pandoc filter cho {{% notice %}} shortcode

report/
├── main.tex                   # khung báo cáo tiếng Việt
├── main_en.tex                # khung báo cáo tiếng Anh
├── ChuongTrinhThucTap.tex / _en.tex   # chèn biểu mẫu D2
├── KetQuaXetTuyen.tex / _en.tex       # chèn biểu mẫu D3
├── TongKet.tex / _en.tex              # khối chữ ký cuối báo cáo
└── Images/hcmut.png           # logo trường (dùng cho trang bìa)

.github/workflows/hugo.yml     # thêm job "build-pdf"
```

*Lưu ý: các file trên là template LaTeX chung (không chứa thông tin cá nhân của ai) lấy từ `fcaj-hcmut-template/`. Nội dung `content/` của repo này (worklog, proposal, blog, self-evaluation...) hoàn toàn không bị thay đổi hay ghi đè bởi bước copy này.*
