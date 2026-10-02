# Hướng dẫn: viết `.py` / `.pyx` và build bằng PyToolsVSCode

Tài liệu này trả lời câu hỏi: **tôi nên tổ chức và viết code Python thế nào để `PyBuildTool` build ra kết quả đúng?** Phần cài đặt và các cờ của từng công cụ xem ở [README](../README.md).

## 1. Công cụ thực sự làm gì

```
 .py / .pyx  --(Cython)-->  C  --(MSVC)-->  .pyd   (mã máy, import được từ Python)
                                               |
 entry point (.py)  ----------+---------------+--(PyInstaller --onefile)--> MyApp.exe
```

- Mọi file `.py` và `.pyx` trong project (trừ **entry point**) được biên dịch thành `.pyd`.
- **Entry point** là file chạy đầu tiên (`MainPyFile` trong cấu hình). Nó được PyInstaller đóng gói cùng các `.pyd` thành một file `.exe`.
- Kết quả nằm trong `build_release\`: `<ExeName>.exe` và các `.pyd` tên sạch (để chia sẻ riêng nếu cần).

**Lưu ý về mức bảo vệ mã nguồn:** phần được biên dịch thành `.pyd` khó đọc ngược hơn mã `.py` thuần, nhưng đây **không phải mã hóa** và không phải bảo vệ tuyệt đối. File `.exe` vẫn chứa trình thông dịch Python và entry point. Đừng nhúng mật khẩu hay khóa bí mật vào code và coi như an toàn.

## 2. Hai chế độ, công cụ tự chọn theo cấu trúc project

| Project có | Công cụ làm gì |
|---|---|
| **Chỉ file `.py`** (không có `.pyx`) | **Tách entry point:** logic thật chuyển sang `build_src\logic.py` rồi biên dịch thành `.pyd`. Entry point chỉ còn vài dòng. |
| **Có ít nhất một file `.pyx`** | **Không tách.** Entry point giữ nguyên là `.py` (đọc được), mọi module còn lại biên dịch thành `.pyd`. |

Ép buộc bằng khóa `AutoSplitEntryPoint` trong cfg thì **khuyến nghị để trống**. Nếu đặt `true` trong khi project đã có `.pyx`, công cụ sẽ từ chối và báo lỗi, để không phá thiết kế Cython bạn viết tay.

## 3. Ví dụ 1: project một file

Cấu trúc:

```
MyApp\
  main.py
```

`main.py`:

```python
import sys


def chao(ten):
    return f"Xin chao, {ten}!"


print(chao("Python"))

if __name__ == "__main__":
    print("Doi so:", sys.argv[1:])
```

Chạy **Py • Build** (lần đầu tạo cfg, chạy lại để build thật). Công cụ tách file thành hai file trong `build_src\`:

`build_src\logic.py` (được biên dịch thành `.pyd`):

```python
import sys


def chao(ten):
    return f"Xin chao, {ten}!"


def main():
    print(chao("Python"))
    print("Doi so:", sys.argv[1:])
```

`build_src\main.py` (entry point, đóng gói vào `.exe`):

```python
from logic import main

if __name__ == "__main__":
    main()
```

Kết quả trong `build_release\`: `MyApp.exe` và `logic.pyd`. File `main.py` gốc của bạn **không bị thay đổi**.

### Quy tắc khi tách entry point

- **Điểm cắt** là hết hàm hoặc class được định nghĩa **cuối cùng**. Mọi thứ phía trên (import, hàm, class, biến) giữ nguyên vị trí trong `logic.py`.
- Code **phía dưới điểm cắt** (các lệnh chạy ở cấp module, và nội dung khối `if __name__ == "__main__":`) được đưa vào hàm `main()` theo đúng thứ tự.
- Biến được gán ở phần dưới được khai báo `global` trong `main()` để các hàm phía trên vẫn dùng được.
- Nếu file đã có tên `main` ở cấp module, công cụ đổi tên hàm thành `__entry_main` để không ghi đè.
- Code chạy ở cấp module **nằm giữa hai hàm** (trước hàm cuối cùng) vẫn nằm trong `logic.py` và chạy lúc `import`, không nằm trong `main()`. Nên đặt code khởi chạy ở **cuối file**, hoặc gói trong một hàm `main()` do bạn tự viết.
- File gốc phải **đúng cú pháp**, nếu không bước tách báo lỗi và dừng.

## 4. Ví dụ 2: nhiều module và `.pyx`

Cấu trúc:

```
MyApp\
  main.py          <- entry point
  tien_ich.py      <- module (thành .pyd)
  tinh_toan.pyx    <- module Cython (thành .pyd)
```

`main.py`:

```python
import tien_ich
import tinh_toan


def main():
    print(tien_ich.chao("Python"))
    print(tinh_toan.tong_binh_phuong(1000))


if __name__ == "__main__":
    main()
```

`tien_ich.py`:

```python
def chao(ten):
    return f"Xin chao, {ten}!"
```

`tinh_toan.pyx`:

```cython
def tong_binh_phuong(int n):
    cdef long long tong = 0
    cdef int i
    for i in range(n):
        tong += i * i
    return tong
```

Vì có `.pyx` nên **không tách**. `main.py` giữ nguyên làm entry point. Kết quả trong `build_release\`: `MyApp.exe`, `tien_ich.pyd`, `tinh_toan.pyd`. Chạy `MyApp.exe` in ra:

```
Xin chao, Python!
332833500
```

## 5. Quy tắc viết `.py`

1. **Đặt tên module không trùng nhau** trong toàn project. Các `.pyd` được gom về cùng một chỗ với tên sạch (`<tên>.pyd`), nên hai file cùng tên ở hai thư mục khác nhau sẽ xung đột.
2. **Import theo kiểu phẳng:** `import tien_ich` hoặc `from tien_ich import chao`. Cấu trúc package (thư mục có `__init__.py`, import tương đối `from . import x`) **chưa được kiểm chứng** với công cụ này, nên chưa khuyến nghị.
3. **Đừng đặt tên file là `logic.py`** nếu project chỉ có `.py` (chế độ tách entry point), vì công cụ tự sinh `logic.py`.
4. **Entry point phải là file `.py`.**
5. **Import tĩnh, đặt ở đầu file.** Công cụ quét các dòng `import x` và `from x import ...` trong các module để truyền cho PyInstaller (`--hidden-import`) vì PyInstaller không đọc được bên trong `.pyd`. Import động (`importlib.import_module(...)`, `__import__(...)`) **không được phát hiện**; khi gặp lỗi `ModuleNotFoundError` trong `.exe`, thêm vào `ExtraPyInstallerArgs` trong cfg, ví dụ `--hidden-import=ten_thu_vien`.
6. **File dữ liệu** (ảnh, JSON, ...) không tự vào `.exe`. Thêm bằng `ExtraPyInstallerArgs`, ví dụ `--add-data "data.json;."`. Với `--onefile`, khi chạy file được giải nén vào thư mục tạm, đường dẫn lấy qua `sys._MEIPASS` thay vì `__file__`.
7. **Ứng dụng giao diện** (Tkinter, PyQt...): đặt `WindowedMode = true` trong cfg để ẩn cửa sổ console.

## 6. Quy tắc viết `.pyx` (Cython)

Công cụ biên dịch `.pyx` với `language_level=3` và `infer_types=True` (Cython tự suy luận kiểu C cho biến cục bộ khi an toàn). Vài điểm cần nhớ:

- `.pyx` là Python mở rộng. Khai báo kiểu C bằng `cdef` giúp tăng tốc rõ rệt ở vòng lặp số học.
- **`cdef` function chỉ dùng được bên trong Cython**, `import` từ Python sẽ không thấy. Muốn gọi từ file `.py` thì khai báo bằng **`def`** (hoặc `cpdef`).
- **Chú ý tràn số:** `int` trong `cdef` là số nguyên 32 bit. Với tổng lớn dùng `long long`, như `tong` ở ví dụ trên.
- File `.pyx` không cần `import` thêm gì để chạy, nhưng nếu dùng thư viện ngoài thì khai báo `import` ở đầu file như bình thường (công cụ quét các dòng này để truyền cho PyInstaller).

## 7. Những thư mục bị bỏ qua khi quét

Công cụ **không** quét (và không biên dịch) file trong thư mục có tên: `__pycache__`, `.git`, `.venv`, `venv`, `env`, `build`, `dist`, `.idea`, `.vscode`, `build_cache`, `build_src`, `build_release`.

> **Cảnh báo:** việc loại trừ kiểm tra **mọi thành phần của đường dẫn đầy đủ**. Nếu cả project nằm bên trong một thư mục tên `build`, `dist`, `env` hoặc `venv` (ví dụ `D:\env\MyApp`), mọi file bị coi là bị loại trừ và công cụ báo không tìm thấy source. Hãy đặt project ở nơi không có thư mục cha mang các tên này.

## 8. Lỗi thường gặp

| Hiện tượng | Nguyên nhân và cách xử lý |
|---|---|
| Báo thiếu "Microsoft Visual C++ 14.0 or greater" lúc biên dịch | Cài **Microsoft C++ Build Tools** (workload Desktop development with C++). Dùng `PyEnvChecker` để kiểm tra. |
| `.exe` chạy báo `ModuleNotFoundError` | Import động hoặc thư viện bị sót: thêm `--hidden-import=...` vào `ExtraPyInstallerArgs`. |
| `.exe` không thấy file dữ liệu | Thêm `--add-data` (mục 5, điểm 6). |
| `.pyd` không import được từ Python khác | `.pyd` gắn với phiên bản Python đã build (hậu tố `cp314` nghĩa là Python 3.14). Dùng đúng phiên bản. |
| Build lỗi vì file bị khóa | `.exe` hoặc `.pyd` đang chạy trong `build_release`. Đóng chương trình rồi build lại. |
| "Không tìm thấy source Python" | Project chưa có `.py`/`.pyx`, hoặc nằm trong thư mục bị loại trừ (mục 7). |
| Muốn build sạch từ đầu | Chạy **Py • Clean**, rồi build lại. |

## 9. Chưa được kiểm chứng

Để không hứa quá những gì đã thử, các trường hợp sau **chưa được kiểm tra kỹ** với công cụ:

- Cấu trúc package nhiều tầng và import tương đối.
- Chương trình dùng `multiprocessing` (PyInstaller thường cần `multiprocessing.freeze_support()` trong script chính, và với chế độ tách entry point thì dòng đó nằm trong code đã được chuyển vào `logic.py`).
- Các framework nặng (PyQt, numpy, ...) có thể cần thêm tham số PyInstaller.

Nếu bạn gặp trường hợp nào ở trên, hãy mở một Issue kèm log, vì thông tin đó giúp cải thiện công cụ.
