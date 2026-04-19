# 📋 Tóm Tắt Kỹ Thuật Tối Ưu Hóa Docker cho NestJS

Tài liệu này ghi lại quá trình tối ưu hóa Docker Image cho ứng dụng NestJS, giúp giảm dung lượng từ **1.98GB** xuống còn **~227MB** (tiết kiệm gần 90% bộ nhớ).

---

## 🚀 1. Hai "Chìa Khóa" Tối Ưu Chính

### A. Sử dụng Base Image Alpine (`node:20-alpine`)
* **Cơ chế:** Thay vì dùng bản Node mặc định (dựa trên Debian/Ubuntu nặng nề), chúng ta dùng **Alpine Linux** - một bản phân phối siêu tối giản chỉ nặng khoảng 5MB.
* **Tại sao hiệu quả:** Nó loại bỏ tất cả các công cụ hệ thống thừa thãi (như trình quản lý gói apt, trình biên dịch, các thư viện mạng không dùng tới).
* **Kết quả:** Giảm ngay lập tức ~300MB - 500MB dung lượng OS nền và tăng tính bảo mật.

### B. Kỹ thuật Multi-stage Build (Xây dựng đa tầng)
* **Cơ chế:** Chia Dockerfile thành nhiều giai đoạn (Stages):
    1.  **Stage Builder:** Dùng image đầy đủ để cài TypeScript, Nest-CLI và biên dịch code `.ts` sang `.js`.
    2.  **Stage Production:** Tạo một image mới "sạch" hoàn toàn, chỉ **COPY** thư mục `dist` và `node_modules` cần thiết sang.
* **Tại sao hiệu quả:** Toàn bộ "rác" phát sinh khi build (source code gốc, compiler, devDependencies, npm cache) đều bị bỏ lại ở Stage cũ, không nằm trong image cuối cùng.

---

## 🛠️ 2. Các Bước Cấu Hình Quan Trọng

### 📝 Tối ưu TypeScript (`tsconfig.json`)
Cần đảm bảo cấu hình chuẩn để thư mục build gọn gàng:
* **`rootDir: "src"`**: Ép TypeScript hiểu `src` là gốc, giúp file chính nằm ngay tại `dist/main.js` thay vì bị lồng trong `dist/src/main.js`.
* **`removeComments: true`**: Xóa chú thích để code JS nhẹ hơn.
* **`sourceMap: false`**: Tắt file bản đồ code (không cần thiết ở môi trường chạy thực tế).

### 🐳 Dockerfile Optimize (Cấu trúc 3 giai đoạn)
Đây là công thức "vàng" để có image siêu nhẹ:

```dockerfile
# STAGE 1: Build source code
FROM node:20-alpine AS builder
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --legacy-peer-deps 
COPY . .
RUN npm run build

# STAGE 2: Lọc thư viện (Production only)
FROM node:20-alpine AS secondary
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --only=production --legacy-peer-deps

# STAGE 3: Image cuối cùng (Siêu nhẹ)
FROM node:20-alpine AS production
WORKDIR /usr/src/app
COPY --from=secondary /usr/src/app/node_modules ./node_modules
COPY --from=builder /usr/src/app/dist ./dist

EXPOSE 3000
CMD ["node", "dist/main"]