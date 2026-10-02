# PyToolsVSCode

Bộ công cụ dòng lệnh cho **Windows** giúp làm việc với project Python trong **VS Code** nhanh hơn: tự build `.py` / `.pyx` thành `.pyd` (Cython) rồi đóng gói `.exe` (PyInstaller), tự build lại khi lưu file, dọn file build, tạo project mới và kiểm tra môi trường. Tất cả chạy bằng một phím tắt trong VS Code (`Ctrl+Shift+B`).

> Đây là bản **phát hành thăm dò ý kiến**. Mình rất muốn nghe bạn thấy công cụ nào hữu ích, công cụ nào thừa, và còn thiếu gì. Xem mục [Góp ý](#góp-ý).

## Bộ công cụ gồm những gì

| Công cụ | Làm gì |
|---|---|
| `PyConfig.exe` | Chạy **một lần**: quét các `Py*.exe` cùng thư mục và thêm chúng thành task trong VS Code. |
| `PyBuildTool.exe` | Build project: `.py` / `.pyx` thành `.pyd` (Cython) rồi thành `.exe` (PyInstaller). Có chế độ dòng lệnh và chế độ giao diện. |
| `PyWatcher.exe` | Theo dõi project, tự gọi `PyBuildTool` khi bạn lưu file `.py` / `.pyx`. |
| `PyClean.exe` | Dọn file build (`build_cache`, `build_src`, `__pycache__`...). Có chế độ xem trước. |
| `PyNewProject.exe` | Tạo project Python mới từ template (có Form, và chạy được từ dòng lệnh). |
| `PyEnvChecker.exe` | Kiểm tra môi trường: Python, thư viện pip, VS Code, MSVC Build Tools, Git. |
| `Internal\EntrySplitter.exe` | Công cụ nội bộ của `PyBuildTool` (tách entry point). **Không chạy trực tiếp**, nhưng phải giữ nguyên trong thư mục `Internal`. |

## Yêu cầu

- Windows 10 / 11, 64-bit.
- [Python](https://www.python.org/downloads/) có trong `PATH`.
- Để build: `pip install cython pyinstaller`.
- Để biên dịch `.pyd`: **Microsoft C++ Build Tools** (workload "Desktop development with C++").
- [VS Code](https://code.visualstudio.com/).

`PyEnvChecker.exe` sẽ kiểm tra giúp bạn các mục trên. Bộ công cụ đã được thử trên Python 3.14 và VS Code 1.140.

## Cài đặt

1. Tải `PyToolsVSCode_v1.0.zip` ở mục [Releases](../../releases).
2. **Chuột phải file zip, chọn Properties, tích Unblock, bấm OK**, rồi mới giải nén. Làm bước này để Windows SmartScreen không chặn từng file `.exe` bên trong.
3. Giải nén vào một thư mục cố định, ví dụ `C:\DevToolsVSCode`. Giữ nguyên thư mục `Internal`.
4. Chạy `PyConfig.exe` **một lần**. Nó thêm các task vào VS Code (xem [PyConfig](#pyconfig)).
5. Mở project Python trong VS Code, bấm `Ctrl+Shift+B`, chọn task `Py` bạn muốn.

> **Lưu ý:** `PyConfig` ghi đường dẫn tuyệt đối của các tool vào VS Code. Nếu sau này bạn **di chuyển thư mục**, hãy chạy lại `PyConfig.exe`.

## Cách dùng nhanh

Mở project Python trong VS Code rồi `Ctrl+Shift+B`:

1. Chọn **Py • Build** (`PyBuildTool`). Lần đầu nó tạo file cấu hình `.pybuild\PyBuildTool.cfg`.
   - Project chỉ có **1 file `.py`**: cấu hình được điền sẵn. Chạy lại task để build thật.
   - Project có **nhiều file `.py`**: hiện một hộp thoại nhỏ để bạn chọn file entry point, lưu xong chạy lại task.
2. Kết quả nằm trong `build_release\`: file `<ExeName>.exe` và các file `.pyd`.
3. Muốn build tự động mỗi khi lưu file: chọn **Py • Watch** (`PyWatcher`).
4. Muốn dọn file build: chọn **Py • Clean** (`PyClean`).

Các công cụ cũng chạy được từ terminal: `PyBuildTool.exe <thư_mục_project>`. Chạy `PyBuildTool.exe` không tham số thì mở giao diện; `PyClean.exe` không truyền thư mục thì dùng thư mục hiện tại.

## Chi tiết từng công cụ

### PyConfig

- Quét các `Py*.exe` **ngay trong thư mục chứa nó** (không quét thư mục con, nên `Internal\EntrySplitter.exe` không bị tạo task).
- Ghi task vào `tasks.json` cấp **User** của VS Code (hỗ trợ VS Code, VS Code Insiders và Cursor).
- **An toàn:** chỉ thêm task của chính nó hoặc cập nhật đường dẫn của task đã có, không sửa task khác (kể cả task Go). Sao lưu bản gốc một lần thành `tasks.json.pyconfig.bak`. Chạy lại nhiều lần không sinh task trùng.
- Thêm tool mới: chép `Py*.exe` vào thư mục rồi chạy lại `PyConfig.exe`.

### PyBuildTool

Quy trình: kiểm tra Python, Cython, PyInstaller; biên dịch các module thành `.pyd`; đóng gói `.exe` bằng PyInstaller.

Cấu hình trong `.pybuild\PyBuildTool.cfg` (dạng `KEY = VALUE`):

| Khóa | Ý nghĩa |
|---|---|
| `ProjectPath` | Thư mục project. |
| `MainPyFile` | File entry point, đường dẫn tương đối so với project. |
| `ExeName` | Tên file `.exe` kết quả (không có `.exe`). |
| `AutoSplitEntryPoint` | Để trống (khuyến nghị) để công cụ tự quyết định; `true` / `false` để ép buộc. |
| `WindowedMode` | `true` để ẩn cửa sổ console (ứng dụng GUI). Mặc định `false`. |
| `CleanTempOnBuild` | Xóa thư mục tạm sau khi build thành công. Mặc định `true`. |
| `ExtraPyInstallerArgs` | Tham số PyInstaller bổ sung. |

Kết quả:

```
<project>\
  build_release\    <- <ExeName>.exe và các file .pyd sạch tên
  build_cache\      <- file .pyd gốc do build_ext sinh ra
  build_src\        <- (nếu tách entry point) file nguồn trung gian
```

Thư mục tạm của quá trình build nằm ở `C:\Temp\pybuildtool_<thời_gian>\`.

### PyWatcher

```
PyWatcher.exe <thư_mục_project> [debounce_ms]
```

Theo dõi file `.py` / `.pyx` (bỏ qua `build_*`, `__pycache__`, `.venv`...). Mặc định chờ 800 ms sau lần lưu cuối rồi mới build. Cần `PyBuildTool.exe` nằm cùng thư mục, và project phải đã có cấu hình (chạy `PyBuildTool` một lần trước).

### PyClean

```
PyClean.exe [<thư_mục_project>] [/dry] [/release] [/all] [/temp]
```

| Cờ | Tác dụng |
|---|---|
| (mặc định) | Xóa `build_cache`, `build_src` và mọi thư mục `__pycache__` trong project. |
| `/dry` | **Chỉ liệt kê**, không xóa gì. Nên chạy trước lần đầu. |
| `/release` | Xóa thêm `build_release` (chứa `.exe` cuối cùng). |
| `/all` | Xóa thêm `build` và `dist` ở gốc project. |
| `/temp` | Xóa thêm `C:\Temp\pybuildtool_*` quá 60 phút. **Đây là rác chung của mọi project**, không riêng project đang dọn. |

**PyClean xóa file thật.** Các biện pháp an toàn:

- Chỉ chạy khi project có `.pybuild\PyBuildTool.cfg`; nếu không thì từ chối và không xóa gì.
- Chỉ xóa **bên trong** thư mục project; không đi theo junction / symlink.
- **Không bao giờ** xóa file `.py`, `.pyx`, `.pyd`, hay thư mục `.venv`, `venv`, `env`, `.git`, `.vscode`, `.idea`, `.pybuild`.
- File đang bị khóa (ví dụ `.exe` đang chạy): báo rõ file nào, dọn tiếp phần còn lại.

Exit code: `0` xong, `1` có mục không xóa được, `2` từ chối hoặc sai tham số.

### PyNewProject

Mở Form để tạo project Python mới từ template, hoặc chạy từ dòng lệnh theo file `.cfg`. Template "gui" **chưa được kiểm chứng kỹ**.

### PyEnvChecker

In ra tình trạng Python, các thư viện pip thường dùng, bản cập nhật có sẵn, VS Code và extension Python, MSVC Build Tools, Git.

## Giới hạn đã biết

- Thư mục tạm của `PyBuildTool` cố định là `C:\Temp` (không theo `%TEMP%`).
- Template "gui" của `PyNewProject` chưa kiểm chứng đầy đủ.
- `PyWatcher` với project **chưa có** file cấu hình chưa được kiểm chứng; hiện nó báo rõ và dừng.
- Chỉ hỗ trợ Windows x64.

## An toàn và cảnh báo của Windows

- File `.exe` **chưa ký số**, nên Windows SmartScreen có thể cảnh báo ("Windows protected your PC"). Bạn có thể bấm *More info*, *Run anyway*, hoặc dùng bước Unblock ở mục [Cài đặt](#cài-đặt).
- Một số phần mềm diệt virus có thể báo nhầm các chương trình có thao tác xóa file hoặc chạy tiến trình con. Nếu bạn muốn chắc chắn, hãy kiểm tra file zip trên [VirusTotal](https://www.virustotal.com/) và đối chiếu mã SHA256:

```
SHA256 (PyToolsVSCode_v1.0.zip): <dán mã SHA256 vào đây>
```

## Gỡ cài đặt

1. Xóa thư mục chứa bộ công cụ.
2. Trong VS Code: `Ctrl+Shift+P`, gõ `Open User Tasks`, xóa các task có nhãn bắt đầu bằng `Py`. Nếu cần, khôi phục `tasks.json` từ file sao lưu `tasks.json.pyconfig.bak` cùng thư mục.

## Góp ý

Mình đang muốn biết:

1. Bạn dùng công cụ nào nhiều nhất? Công cụ nào bạn không bao giờ dùng?
2. Quy trình build `.py` thành `.pyd` rồi `.exe` có hợp với cách bạn làm việc không?
3. Bạn còn thiếu tính năng gì (ví dụ: thêm template, hỗ trợ `requirements.txt`, ký số file `.exe`...)?
4. Bạn gặp lỗi hoặc cảnh báo gì khi cài đặt?

Hãy mở một [Issue](../../issues) hoặc viết trong [Discussions](../../discussions) (đính kèm log của công cụ nếu có lỗi). Cảm ơn bạn đã thử!

## Giấy phép

Bản biên dịch được **phát hành miễn phí** để sử dụng và chia sẻ lại **nguyên vẹn** (không chỉnh sửa, kèm file này). Phần mềm được cung cấp "nguyên trạng", **không bảo hành**; tác giả không chịu trách nhiệm về bất kỳ thiệt hại hoặc mất mát dữ liệu nào phát sinh khi sử dụng. Mã nguồn không được công bố.

Tác giả: **Kieu Manh**.

---

## English summary

**PyToolsVSCode** is a set of Windows command-line tools for Python projects in VS Code: build `.py`/`.pyx` to `.pyd` (Cython) and `.exe` (PyInstaller) with `PyBuildTool`, auto-rebuild on save with `PyWatcher`, safe cleanup with `PyClean` (`/dry` preview), project scaffolding with `PyNewProject`, environment checks with `PyEnvChecker`, and one-time VS Code task setup with `PyConfig`. Download the zip from Releases, unblock it (right-click, Properties, Unblock), extract it keeping the `Internal` folder, run `PyConfig.exe` once, then use `Ctrl+Shift+B` in VS Code. Binaries are unsigned, so SmartScreen may warn. Feedback is very welcome via Issues or Discussions.
