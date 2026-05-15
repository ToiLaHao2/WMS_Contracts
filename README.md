# WMSS Contracts (gRPC Protobufs)

Kho lưu trữ này đóng vai trò là **"Hợp đồng Giao tiếp" (Single Source of Truth)** cho toàn bộ hệ thống mô phỏng tự động hóa nhà kho đa ngôn ngữ (WMSS). 

Nó chứa tất cả các tệp định nghĩa Protocol Buffers (`.proto`) dùng để giao tiếp đồng bộ (gRPC) giữa các Microservices.

## 🌟 Tầm quan trọng của kho lưu trữ này

Trong kiến trúc Polyglot Microservices (sử dụng nhiều ngôn ngữ lập trình khác nhau), việc đảm bảo tính nhất quán về cấu trúc dữ liệu khi giao tiếp là cực kỳ quan trọng. Bằng cách tách các tệp `.proto` ra thành một kho lưu trữ riêng biệt (được đính kèm vào các dự án khác dưới dạng Git Submodule):

1. **Đồng nhất dữ liệu:** Đảm bảo Node.js, Python và Golang đều "nói chung một ngôn ngữ". Khi cấu trúc dữ liệu thay đổi, tất cả các dịch vụ đều được cập nhật đồng bộ.
2. **Contract-First Development:** Buộc các developer phải thống nhất Interface (Đầu vào/Đầu ra) trước khi bắt tay vào code logic ở bất kỳ ngôn ngữ nào.
3. **Quản lý phiên bản độc lập:** Việc cập nhật giao thức không làm ảnh hưởng đến mã nguồn logic của từng dự án.

---

## 🏛️ Kiến trúc Giao tiếp gRPC trong hệ thống

Kho lưu trữ này định nghĩa các luồng gọi (RPC calls) sau:

- **WMS (Node.js) ↔ MES (Python):** 
  - WMS gọi MES để lấy thông tin truy vấn hoặc kết quả mô phỏng (Preview/Query).
  - *Lưu ý:* WMS đẩy lệnh nghiệp vụ chính xuống MES thông qua Kafka (Async), không qua gRPC.
  
- **MES (Python) ↔ AGV Control (Golang):**
  - MES sau khi tính toán xong gửi **Execution Plan** (Lộ trình, Danh sách tác vụ vật lý) xuống cho tầng Golang thực thi.
  - Golang có thể gọi ngược lại MES để xin tính toán lại đường đi (Re-route) khi gặp vật cản bất ngờ.

---

## 🚀 Hướng dẫn tích hợp cho các ngôn ngữ

Kho lưu trữ này được tích hợp vào các dự án chính thông qua lệnh `git submodule`.

### 1. Dành cho Tầng WMS (Node.js / TypeScript)
Node.js đọc trực tiếp tệp `.proto` lúc runtime thông qua tính năng "Dynamic Loading".
- **Thư viện:** `@grpc/grpc-js`, `@grpc/proto-loader`
- **Cách dùng:** Không cần biên dịch mã nguồn. Sử dụng `protoLoader.loadSync()` trỏ thẳng đến tệp `.proto` trong thư mục Submodule.

### 2. Dành cho Tầng MES (Python)
Python cần biên dịch tệp `.proto` thành mã Python tĩnh (Static Generation) để tận dụng tính năng kiểm tra kiểu (Type-checking).
- **Thư viện:** `grpcio`, `grpcio-tools`
- **Cách dùng:** Chạy lệnh biên dịch sau mỗi lần tệp `.proto` thay đổi:
  ```bash
  python -m grpc_tools.protoc -I./libs/core/contracts/Warehouse_management_simulation_Contracts \
      --python_out=./libs/core/contracts \
      --grpc_python_out=./libs/core/contracts \
      ./libs/core/contracts/Warehouse_management_simulation_Contracts/mes.proto
  ```
- *Mẹo:* Đừng quên chỉnh sửa import path trong tệp `_grpc.py` được sinh ra để fix lỗi đường dẫn tương đối của Python.

### 3. Dành cho Tầng AGV Control (Golang)
Golang yêu cầu biên dịch mạnh (Strongly-typed Generation).
- **Công cụ:** `protoc`, `protoc-gen-go`, `protoc-gen-go-grpc`
- **Cách dùng:**
  ```bash
  protoc --go_out=. --go-grpc_out=. path/to/mes.proto
  ```
- *Lưu ý:* Các tệp `.proto` trong kho lưu trữ này đều đã được khai báo sẵn thuộc tính `option go_package = "github.com/devil/wmss/contracts/...";` để hỗ trợ Golang build chính xác module path.

---

## 📜 Quy tắc đóng góp (Contributing)

1. **KHÔNG** sửa trực tiếp tệp `.proto` bên trong các thư mục Submodule của từng dự án.
2. Để thay đổi contract:
   - Thay đổi trong repository gốc (kho lưu trữ này).
   - Commit và Push lên nhánh `main`.
   - Vào từng dự án Node.js / Python / Go, kéo (Pull) phiên bản Submodule mới nhất về và build lại (nếu cần).
3. Đảm bảo tính tương thích ngược (Backward Compatibility). Hạn chế xóa/đổi tên trường (Field) đã tồn tại; thay vào đó, hãy thêm trường mới nếu cần thiết.
