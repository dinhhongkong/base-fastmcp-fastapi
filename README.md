# base-fastmcp-fastapi

Hướng dẫn cài môi trường trên Linux (Ubuntu/Debian, Bash): dùng `pyenv` quản lý phiên bản Python và `uv` quản lý môi trường ảo, dependency.

## 1. Cài pyenv

Cài các gói cần thiết để biên dịch Python:

```bash
sudo apt update
sudo apt install -y build-essential git curl libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev libncursesw5-dev \
  xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
```

Cài pyenv (nếu chưa có):

```bash
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
```

Thêm các dòng sau vào cuối `~/.bashrc` và file cấu hình login Bash đang dùng (`~/.bash_profile`, hoặc `~/.profile` nếu không có `~/.bash_profile`):

```bash
export PYENV_ROOT="$HOME/.pyenv"
[[ -d "$PYENV_ROOT/bin" ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - bash)"
```

Mở terminal mới, sau đó kiểm tra:

```bash
pyenv --version
```

Với bản phân phối Linux khác hoặc shell khác, xem [hướng dẫn cài pyenv](https://github.com/pyenv/pyenv#installation).

## 2. Cài đúng Python cho dự án

Chạy các lệnh sau trong thư mục gốc repository:

```bash
cat .python-version
pyenv install -s "$(cat .python-version)"
python --version
pyenv which python
```

Dự án hiện ghi `3.12` trong `.python-version`, còn `pyproject.toml` yêu cầu Python `>=3.12`. Với giá trị `3.12`, `pyenv install` chọn bản vá mới nhất thuộc nhánh 3.12 mà bản pyenv đang cài biết đến. Khi đứng trong repository, pyenv tự đọc `.python-version` để chọn Python.

Nếu nhóm cần cố định cả phiên bản vá, chạy `pyenv local 3.12.X` (thay `X` bằng số bản vá đã thống nhất và cài bằng `pyenv install -s 3.12.X` trước). Lệnh này cập nhật `.python-version`; commit file đó để các thành viên dùng cùng phiên bản. Không cần đổi Python toàn hệ thống bằng `pyenv global`.

## 3. Cài uv và đồng bộ dependency

Cài uv theo [hướng dẫn chính thức](https://docs.astral.sh/uv/getting-started/installation/):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source "$HOME/.local/bin/env"
uv --version
```

Tại thư mục gốc repository, tạo môi trường bằng đúng Python của pyenv và cài dependency từ lockfile đã commit:

```bash
uv sync --locked --python "$(pyenv which python)"
uv run --locked python --version
uv run --locked python -c 'import sys; print(sys.executable); print(sys.base_prefix)'
```

`uv` tạo `.venv` nếu chưa có. `sys.executable` trỏ vào `.venv`, còn `sys.base_prefix` cho biết Python nền được dùng. Nếu đã có `.venv` từ một phiên bản Python khác, hãy đổi tên thư mục đó rồi chạy lại lệnh sync ở trên.

Sau khi pull code hoặc khi cần đồng bộ môi trường trong quá trình phát triển:

```bash
uv sync
```

`uv sync` đồng bộ `.venv` với dependency của dự án, có thể cập nhật `uv.lock` khi cấu hình thay đổi và mặc định loại các package không thuộc tập dependency được chọn. Lệnh không tự nâng mọi package lên bản mới nhất.

Dùng `uv sync --locked` khi muốn giữ lockfile nguyên trạng (ví dụ sau khi clone hoặc trong CI). Nếu lockfile không khớp `pyproject.toml`, lệnh báo lỗi; người thay đổi dependency cần chạy `uv sync`, kiểm tra và commit lockfile mới. Xem [cơ chế lock và sync](https://docs.astral.sh/uv/concepts/projects/sync/).

Không bắt buộc activate môi trường khi dùng `uv run`. Nếu muốn gọi `python` trực tiếp trong môi trường dự án:

```bash
source .venv/bin/activate
python --version
deactivate
```

## 4. Thêm và xóa thư viện bằng uv

Các lệnh dưới đây là ví dụ; chỉ chạy lệnh tương ứng với thư viện cần dùng:

```bash
# Thêm dependency chạy ứng dụng
uv add httpx

# Chỉ định khoảng phiên bản hoặc cố định phiên bản
uv add 'httpx>=0.28,<1.0'
uv add 'httpx==0.28.1'

# Thêm dependency chỉ phục vụ phát triển
uv add --dev pytest ruff

# Xóa dependency
uv remove httpx
uv remove --dev ruff
```

`uv add` / `uv remove` cập nhật `pyproject.toml`, `uv.lock` và đồng bộ môi trường. Nhóm `dev` được cài mặc định khi sync; dùng `uv sync --locked --no-dev` nếu không cần nhóm này. Xem [quản lý dependency](https://docs.astral.sh/uv/concepts/projects/dependencies/).

Để ghi nhận dependency cho cả nhóm, dùng `uv add` thay vì chỉ cài bằng `pip install` hoặc `uv pip install`.

## 5. Cập nhật phiên bản dependency

```bash
# Nâng một package trong giới hạn phiên bản của pyproject.toml
uv lock --upgrade-package httpx
uv sync

# Hoặc nâng tất cả package trong giới hạn đã khai báo
uv lock --upgrade
uv sync

# Kiểm tra lockfile có khớp cấu hình không
uv lock --check
```

Nếu dependency đang khóa bằng `==`, muốn đổi phiên bản phải đổi ràng buộc, ví dụ `uv add 'httpx==0.28.1'`. Cờ upgrade không vượt qua giới hạn phiên bản trong `pyproject.toml`.

## 6. Các file cần quản lý bằng Git

| File/thư mục | Vai trò | Commit |
| --- | --- | --- |
| `.python-version` | Phiên bản Python dự án chọn | Có |
| `pyproject.toml` | Cấu hình dự án và ràng buộc dependency | Có |
| `uv.lock` | Phiên bản dependency đã resolve | Có |
| `.venv/` | Môi trường ảo trên từng máy | Không |

Sau khi thêm, xóa hoặc cập nhật thư viện, kiểm tra diff và commit cả `pyproject.toml` lẫn `uv.lock` nếu chúng thay đổi. Thành viên khác chỉ cần pull code rồi chạy `uv sync --locked`.
