---
name: threejs-3d-exhibition
description: "Hướng dẫn kiến trúc, thiết kế vật liệu và lập trình không gian triển lãm bảo tàng ảo 3D bằng Three.js trong React/Next.js. Sử dụng khi xây dựng phòng trưng bày 3D, showroom ảo, bảo tàng số, tối ưu vật liệu pha lê (MeshPhysicalMaterial), ánh sáng spotlight, joystick cảm ứng mobile, và nạp mô hình GLB/GLTF động."
metadata:
  author: ant-gravity
  version: "1.0.0"
---

# Kỹ Năng Thiết Kế & Xây Dựng Phòng Triển Lãm 3D (Three.js + Next.js)

Tài liệu này đúc kết toàn bộ kiến thức thực chiến, kiến trúc mã nguồn và giải pháp kỹ thuật cao cấp từ dự án Triển Lãm 3D PAST (Công An Phường Tân Hưng). Dùng làm tiêu chuẩn cho mọi dự án Three.js tương lai.

---

## 1. Kiến Trúc Vòng Lặp & Tích Hợp React / Next.js

### 1.1 Quản Lý State Không Re-render (`stateRef`)
Trong Three.js, việc cập nhật tọa độ camera ở 60 FPS qua `useState` sẽ gây re-render toàn bộ DOM và phá hủy hiệu năng.
- **Giải pháp:** Dùng `useRef` lưu toàn bộ trạng thái chuyển động, góc quay yaw/pitch, cờ kéo chuột và phím bấm:
  ```javascript
  const stateRef = useRef({
    posX: 0,
    posZ: 2,
    rotY: 0,
    rotX: 0,
    keys: {},
    mouseDrag: false,
    joyX: 0,
    joyY: 0,
    interactionPrompt: null
  });
  ```
- **Xử lý gõ phím tiếng Việt (Telex Layout Immunity):** Kiểm tra đồng thời cả `e.key` lẫn `e.code` (`e.code === 'KeyW'` không bao giờ bị ảnh hưởng bởi bộ gõ Telex Unikey/EVKey).

### 1.2 Dọn Dẹp Tài Nguyên Triệt Để (Zero Memory Leak)
Khi unmount hoặc đổi route, phải duyệt đệ quy giải phóng mọi Geometry, Material và Texture:
```javascript
const disposeNode = (node) => {
  if (node.geometry) node.geometry.dispose();
  if (node.material) {
    if (Array.isArray(node.material)) {
      node.material.forEach(m => {
        if (m.map) m.map.dispose();
        m.dispose();
      });
    } else {
      if (node.material.map) node.material.map.dispose();
      node.material.dispose();
    }
  }
};
```

---

## 2. Kỹ Nghệ Vật Liệu Bảo Tàng Cao Cấp (Photorealistic Materials)

### 2.1 Mặt Bàn Đá Obsidian Đen Bóng
```javascript
const cabMat = new THREE.MeshPhysicalMaterial({ 
  color: 0x070b14, 
  roughness: 0.04, 
  metalness: 0.45,
  clearcoat: 1.0,
  clearcoatRoughness: 0.02,
  reflectivity: 0.95
});
```

### 2.2 Vòm Kính Pha Lê Quang Học (Crystal Bell Jar)
Dùng `MeshPhysicalMaterial` với độ truyền quang cao và chiết suất thủy tinh thực tế:
```javascript
const bellJarMat = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  transparent: true,
  opacity: 0.92,
  transmission: 0.94,
  ior: 1.54,
  roughness: 0.02,
  metalness: 0.03,
  clearcoat: 1.0,
  clearcoatRoughness: 0.02,
  reflectivity: 0.98,
  thickness: 0.07,
  side: THREE.DoubleSide,
  depthWrite: false
});
```

### 2.3 Viền Mạ Vàng 24K & Dải Đèn LED Neon
```javascript
const goldTrimMat = new THREE.MeshStandardMaterial({ 
  color: 0xf5d061, 
  metalness: 0.95, 
  roughness: 0.08 
});
const ledNeonMat = new THREE.MeshBasicMaterial({ color: 0x38bdf8 });
```

---

## 3. Quy Chuẩn Bố Trí Không Gian 1 Tầng & So Le Chiều Sâu

### 3.1 Thiết Kế Bàn Phẳng 1 Tầng (Single-tier Flat Tabletop)
- Độ cao chuẩn mặt bàn: $Y = 0.88\text{m}$.
- Đĩa mạ vàng & đệm nhung: $Y = 0.89\text{m}$.
- Chân đế chuông pha lê: $Y = 0.90\text{m}$.
- Tâm mẫu vật lơ lửng: $Y = 1.03\text{m}$.

### 3.2 Bố Trí So Le Chiều Sâu (Staggered Placement)
Mỗi tủ chứa 12 mẫu vật: 6 hàng trước và 6 hàng sau trên cùng một mặt bàn phẳng, lệch tọa độ $0.55\text{m} - 0.75\text{m}$ để người xem quan sát từ lối đi không bị che khuất tầm nhìn.

### 3.3 Treo Tranh/Áp Phích Đa Diện Tường Theo Vector Pháp Tuyến
Tự động tính góc hướng ra khỏi tường dựa trên góc xoay Euler:
```javascript
const normal = new THREE.Vector3(0, 0, 1).applyEuler(new THREE.Euler(rx, ry, rz));
surface.position.copy(frame.position).addScaledVector(normal, 0.046);

// Spotlight rọi tranh lùi về phía trước mặt tranh 1.4m
const spotPos = new THREE.Vector3().copy(frame.position).addScaledVector(normal, 1.4);
spotPos.y = 3.8;
posterSpot.position.copy(spotPos);
```

---

## 4. Tối Ưu Hóa & Điều Khiển Đa Nền Tảng (Mobile + Desktop)

### 4.1 Virtual Joystick Trên Mobile
- Bắt sự kiện Touch thủ công với `{ passive: false }` để chặn triệt để hiện tượng cuộn trang mặc định của trình duyệt điện thoại.
- Giới hạn bán kính ngón tay bằng `maxJoystickRadius` và chuyển đổi thành hệ vector lực `joyX`, `joyY` từ `-1.0` đến `1.0`.

### 4.2 Chặn Cuộn Lỗi và Phóng To Không Mong Muốn
- Chặn cử chỉ pinch-to-zoom hai ngón tay trên toàn màn hình.
- Chặn phóng to khi double-tap nhanh trên Safari iOS.
- Cho phép cuộn cảm ứng tự nhiên đối với các thành phần Modal có nội dung dài.

---

## 5. Xử Lý Mô Hình 3D Động (GLTF/GLB) & Hệ Số Thu Phóng (Scale)

### 5.1 Chuẩn Hóa Khung Bao (Bounding Box Normalization)
Mỗi mô hình GLB tải lên từ người dùng có kích thước và gốc tọa độ khác nhau. Bắt buộc chuẩn hóa đưa tâm về `(0, 0, 0)` và scale về kích thước khung bảo tàng `targetSize`:
```javascript
const box = new THREE.Box3().setFromObject(model);
const center = new THREE.Vector3();
box.getCenter(center);
model.position.sub(center);

const size = new THREE.Vector3();
box.getSize(size);
const maxDim = Math.max(size.x, size.y, size.z);
const targetSize = 0.22;
const scaleMultiplier = ex.scale || 1.0;

if (maxDim > 0) {
  const modelScale = (targetSize / maxDim) * scaleMultiplier;
  model.scale.set(modelScale, modelScale, modelScale);
}
```

---

## 6. Checklist Khi Phát Triển Phòng Triển Lãm 3D Mới
1. [ ] Kiểm tra camera không nằm sát vách hoặc đâm xuyên bàn lúc khởi tạo.
2. [ ] Khởi tạo sàn và trần với `MeshBasicMaterial` hoặc thiết lập `maxAnisotropy` cho CanvasTexture để khử răng cưa góc xiên.
3. [ ] Đảm bảo kính chuông pha lê có `depthWrite: false` để nhìn thấy mẫu vật bên trong không bị lỗi Z-fighting.
4. [ ] Bố trí hộp va chạm (`checkCollisions`) có độ rộng đệm `0.2m` tương đương kích thước thân người chơi.
5. [ ] Đồng bộ hóa dữ liệu qua phiên bản cache `DATA_VERSION` để client luôn nhận bản cập nhật mới nhất.
