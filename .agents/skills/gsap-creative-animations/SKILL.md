---
name: gsap-creative-animations
description: "Hướng dẫn chuyên sâu và chuẩn mực kỹ thuật sử dụng GSAP (GreenSock Animation Platform) trong React, Next.js và Three.js. Áp dụng khi tạo hiệu ứng chuyển cảnh 3D mượt mà (camera transitions, timeline), hiệu ứng cuộn trang ScrollTrigger, điều khiển timeline phức tạp, và tối ưu hóa hiệu năng animation 60-120 FPS."
metadata:
  author: ant-gravity
  version: "1.0.0"
---

# Kỹ Năng Lập Trình GSAP Animation (React, Next.js & Three.js)

Tài liệu này đúc kết toàn bộ kiến thức, phương pháp và best practices chuyên sâu khi ứng dụng **GSAP** (GreenSock Animation Platform) trong các ứng dụng web tương tác hiện đại, đặc biệt là khi kết hợp cùng **Three.js** và **React/Next.js**.

---

## 1. Nguyên Tắc Cốt Lõi Của GSAP Trong React / Next.js

### 1.1 Sử Dụng `@gsap/react` Hook (`useGSAP`)
Trong React 18 & 19 (StrictMode), các effect chạy 2 lần khi mount. Nếu khởi tạo GSAP thông thường trong `useEffect`, các animation sẽ bị đè lặp hoặc gây rò rỉ bộ nhớ.
- **Tiêu chuẩn:** Luôn dùng `useGSAP` với scoping container:
  ```javascript
  import { useRef } from 'react';
  import gsap from 'gsap';
  import { useGSAP } from '@gsap/react';

  gsap.registerPlugin(useGSAP);

  export default function AnimatedComponent() {
    const containerRef = useRef(null);

    useGSAP(() => {
      // Mọi selector bên trong tự động scoped vào containerRef
      gsap.from('.box', { 
        y: 60, 
        opacity: 0, 
        duration: 0.8, 
        stagger: 0.1, 
        ease: 'power3.out' 
      });
    }, { scope: containerRef });

    return (
      <div ref={containerRef}>
        <div className="box">Item 1</div>
        <div className="box">Item 2</div>
      </div>
    );
  }
  ```
- **Tự động Cleanup:** `useGSAP` tự động gọi `.revert()` khi unmount, dọn sạch mọi timeline và style nội suy.

---

## 2. Kết Hợp GSAP Với Three.js (Chuyển Động 3D Mượt Mà)

### 2.1 Hoạt Hóa Camera 3D (Smooth Camera Orbit / Fly-to)
Thay vì đổi góc camera đột ngột, dùng GSAP nội suy vector vị trí và góc nhìn:
```javascript
const focusOnExhibit = (camera, targetPosition, lookAtTarget) => {
  const tl = gsap.timeline({
    defaults: { duration: 1.6, ease: 'power2.inOut' }
  });

  // 1. Bay camera tới vị trí quan sát trước mẫu vật
  tl.to(camera.position, {
    x: targetPosition.x,
    y: targetPosition.y + 0.3,
    z: targetPosition.z + 1.2,
    onUpdate: () => {
      camera.lookAt(lookAtTarget.x, lookAtTarget.y, lookAtTarget.z);
    }
  });
};
```

### 2.2 Hoạt Hóa Thuộc Tính Vật Liệu (Materials & Lighting)
Nội suy độ trong suốt, màu sắc hoặc cường độ sáng:
```javascript
// Hiệu ứng làm mờ lồng kính khi kích hoạt tương tác
gsap.to(bellJarMat, {
  opacity: 0.4,
  transmission: 0.98,
  duration: 0.6,
  ease: 'sine.out'
});

// Hiệu ứng đèn spotlight bừng sáng (SpotLight intensity)
gsap.to(spotLight, {
  intensity: 8.5,
  duration: 0.4,
  yoyo: true,
  repeat: 1,
  ease: 'power1.inOut'
});
```

### 2.3 Đồng Bộ Tốc Độ Xoay 3D Bằng Timeline
```javascript
// Xoay vật thể 360 độ liên tục với khả năng pause / resume
const rotationTween = gsap.to(specimenMesh.rotation, {
  y: `+=${Math.PI * 2}`,
  duration: 12,
  repeat: -1,
  ease: 'none',
  paused: false
});

// Dừng khi hover chuột, tiếp tục khi rời chuột
mountElement.addEventListener('mouseenter', () => rotationTween.pause());
mountElement.addEventListener('mouseleave', () => rotationTween.play());
```

---

## 3. Quản Lý Timeline Phức Tạp (Sequence Choreography)

### 3.1 Dùng Nhãn (Labels) & Vị Trí Tương Đối
```javascript
const introTimeline = gsap.timeline();

introTimeline
  .addLabel('start')
  .from('.hud-header', { y: -40, opacity: 0, duration: 0.6 }, 'start')
  .from('.canvas-container', { scale: 1.05, opacity: 0, duration: 1.2 }, 'start')
  .addLabel('contentIn', '-=0.4')
  .from('.sidebar-left', { x: -60, opacity: 0, duration: 0.8 }, 'contentIn')
  .from('.footer-bar', { y: 40, opacity: 0, duration: 0.8 }, 'contentIn');
```

### 3.2 Tối Ưu Hóa Hiệu Năng (60 - 120 FPS)
- **Chỉ animate thuộc tính GPU:** `transform` (`x`, `y`, `scale`, `rotation`) và `opacity`. Tránh tuyệt đối animate `top`, `left`, `width`, `height`, `margin` vì sẽ kích hoạt layout reflow.
- **Thêm `force3D: true`:** Ép trình duyệt dựng layer riêng biệt trên GPU.
- **Dùng `gsap.ticker`:** Khi cần đồng bộ animation GSAP với render loop của Three.js:
  ```javascript
  gsap.ticker.add((time, deltaTime, frame) => {
    renderer.render(scene, camera);
  });
  ```

---

## 4. ScrollTrigger: Cuộn Trang Điều Khiển Không Gian 3D

Khi kết hợp bảo tàng ảo với trang thông tin dạng cuộn (Scrollytelling):
```javascript
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

// Camera lướt qua từng tủ trưng bày theo bước cuộn
gsap.timeline({
  scrollTrigger: {
    trigger: '#exhibition-section',
    start: 'top top',
    end: '+=3000',
    scrub: 1.2, // Chuyển động bám theo cuộn chuột mượt mà
    pin: true   // Khóa khung nhìn 3D trong khi cuộn nội dung
  }
})
.to(camera.position, { x: -4.95, z: -0.75 }) // Tới Tủ 1
.to(camera.position, { x: 4.95, z: -0.75 })  // Tới Tủ 2
.to(camera.position, { x: 0.0, z: 6.45 });   // Tới Tủ 3
```

---

## 5. Checklist Tránh Lỗi Phổ Biến (Common Pitfalls)
1. [ ] **StrictMode Duplicate:** Luôn dùng `useGSAP` hoặc gọi `ctx.revert()` trong cleanup của `useEffect`.
2. [ ] **CSS Flash (FOUC):** Đặt `visibility: hidden` hoặc `opacity: 0` trên CSS gốc cho các phần tử dùng `gsap.from()`, sau đó dùng `gsap.set(el, { visibility: 'visible' })` trước khi chạy animation.
3. [ ] **Z-fighting & Matrix Update:** Khi dùng GSAP animate Three.js camera hoặc mesh, đảm bảo cập nhật matrix nếu đang tắt `matrixAutoUpdate`.
4. [ ] **Kill Tweens:** Trước khi bắt đầu một animation mới trên cùng một object, gọi `gsap.killTweensOf(target)` để chống xung đột tween.
