# Session 10 - Bài 1: Báo cáo kỹ thuật phân tích kiến trúc Container vs Virtual Machine

> **Bối cảnh:** doanh nghiệp đang chạy mỗi ứng dụng trên một VPS riêng (mỗi VPS là một máy ảo). Báo cáo này phân tích lý do nên chuyển sang đóng gói và chạy ứng dụng bằng Docker container, dựa trên sự khác biệt về kiến trúc và số liệu thực tế đo trên máy chủ `wean2607`.

## 1. Kiến trúc: ảo hoá phần cứng và ảo hoá hệ điều hành

```text
        VIRTUAL MACHINE                         CONTAINER
 ┌───────────┐ ┌───────────┐          ┌───────────┐ ┌───────────┐
 │   App A   │ │   App B   │          │   App A   │ │   App B   │
 │ Libs/Bins │ │ Libs/Bins │          │ Libs/Bins │ │ Libs/Bins │
 │ Guest OS  │ │ Guest OS  │          └─────┬─────┘ └─────┬─────┘
 │ (kernel)  │ │ (kernel)  │                │             │
 └─────┬─────┘ └─────┬─────┘          ┌─────┴─────────────┴─────┐
 ┌─────┴─────────────┴─────┐          │ Docker Engine (dockerd, │
 │ Hypervisor (KVM, ESXi…) │          │   containerd, runc)     │
 ├─────────────────────────┤          ├─────────────────────────┤
 │        Host OS          │          │  Host OS + Linux kernel │
 ├─────────────────────────┤          │      (DÙNG CHUNG)       │
 │        Phần cứng        │          ├─────────────────────────┤
 └─────────────────────────┘          │        Phần cứng        │
                                      └─────────────────────────┘
```

- **Máy ảo (VM)** ảo hoá **phần cứng**. Hypervisor giả lập CPU, RAM, ổ đĩa, card mạng; trên đó mỗi VM cài và khởi động **một hệ điều hành đầy đủ với kernel riêng**.
- **Container** ảo hoá **hệ điều hành**. Không có Guest OS: mọi container dùng chung kernel của máy chủ. Thực chất container chỉ là **một tiến trình Linux bình thường** được kernel "bao" lại bằng Namespaces (giới hạn những gì nó *nhìn thấy*) và cgroups (giới hạn những gì nó *được dùng*).

## 2. Bảng so sánh VM và Container theo 5 tiêu chí

| Tiêu chí | Virtual Machine | Container | Ghi chú thực tế |
|---|---|---|---|
| **Dung lượng Image** | Vài GB đến hàng chục GB (gồm cả OS, kernel, driver) | Vài MB đến vài trăm MB (chỉ ứng dụng và thư viện cần thiết) | Image `my-html-app:v1` (bài 3, Session 09) chỉ có **26.3 MB** nội dung; `hello-world` chưa tới 20 KB |
| **Thời gian khởi động** | 30 giây đến vài phút (phải boot BIOS, kernel, systemd) | Dưới 1 giây (chỉ là khởi một tiến trình) | `docker run` container Nginx ở Session 09 lên trạng thái `Up` ngay lập tức |
| **Hao phí OS** | Mỗi VM tốn sẵn vài trăm MB đến vài GB RAM và CPU cho Guest OS, kể cả khi ứng dụng không làm gì | Gần như bằng 0: không có Guest OS, chỉ tốn tài nguyên của chính tiến trình ứng dụng | Trên VPS 1 CPU / 1.9 GiB RAM, chạy nhiều VM lồng nhau là bất khả thi, nhưng chạy nhiều container thì được |
| **Mức độ cô lập** | Rất mạnh: kernel riêng, ranh giới là phần cứng ảo do hypervisor bảo vệ | Trung bình đến mạnh: cô lập ở mức tiến trình bằng Namespaces, cgroups, seccomp, AppArmor; **dùng chung kernel** nên lỗ hổng kernel ảnh hưởng mọi container | Container Linux không chạy được trên kernel Windows và ngược lại |
| **Khả năng mở rộng** | Chậm và tốn kém: mỗi bản sao mới phải cấp VM, cài OS, cấu hình | Nhanh và rẻ: nhân bản bằng `docker run` từ cùng một image, mật độ hàng trăm container trên một host | Một image chạy được ở mọi nơi có Docker: máy dev, CI, server, cloud |

**Kết luận phần 2:** VM vẫn cần cho những trường hợp đòi hỏi cô lập tuyệt đối hoặc chạy hệ điều hành khác. Trong thực tế, hai công nghệ thường kết hợp: chính VPS của chúng ta là một VM, và Docker chạy bên trong nó.

## 3. Cơ chế cô lập: Linux Namespaces

Namespace làm cho một tiến trình **chỉ nhìn thấy** một "lát" tài nguyên của hệ thống, như thể nó đang chạy trên một máy riêng.

| Namespace | Cô lập cái gì | Hệ quả trong container |
|---|---|---|
| **PID** | Cây tiến trình | Tiến trình chính trong container có **PID 1** và không thấy tiến trình của host hay container khác. Đây là lý do `ENTRYPOINT` nên dùng exec form để `java` nhận được `SIGTERM` |
| **NET** | Card mạng, IP, bảng định tuyến, cổng | Mỗi container có `eth0` và IP riêng trong mạng bridge. Hai container cùng nghe cổng 80 không đụng nhau; muốn truy cập từ ngoài phải map cổng (`-p 8080:80`) |
| **MNT** | Điểm gắn kết (mount points), hệ thống file | Container thấy root filesystem riêng dựng từ các layer của image (overlayfs). Volume như `easy-volume:/data` được gắn thêm vào đúng điểm mount này |
| **UTS** | Hostname | Container có hostname riêng (mặc định là Container ID) |
| **IPC** | Shared memory, semaphore | Container không đọc được bộ nhớ chia sẻ của tiến trình khác |
| **USER** | UID/GID | Có thể ánh xạ root trong container thành user không đặc quyền ngoài host |

## 4. Quản lý tài nguyên: Control Groups (cgroups)

Namespace chỉ giới hạn tầm nhìn; nếu thiếu cgroups, một container vẫn có thể ngốn hết CPU và RAM của cả máy. **cgroups** cho phép kernel **đo đếm và giới hạn tài nguyên** cho từng nhóm tiến trình:

| Tài nguyên | Cách giới hạn với Docker | Ý nghĩa |
|---|---|---|
| CPU | `--cpus=0.5`, `--cpu-shares` | Container chỉ dùng tối đa nửa nhân CPU, không làm nghẽn ứng dụng khác |
| RAM | `--memory=512m`, `--memory-swap` | Vượt giới hạn thì kernel kill tiến trình trong container (OOM), không làm sập cả host |
| Số tiến trình | `--pids-limit=100` | Chống fork bomb |
| I/O ổ đĩa | `--device-read-bps`, `--device-write-bps` | Hạn chế một container chiếm hết băng thông đĩa |

Ví dụ:

```bash
docker run -d --name web --cpus=0.5 --memory=256m nginx:alpine
docker stats web --no-stream    # cột MEM LIMIT hiển thị 256MiB
```

Máy chủ của chúng ta dùng **Cgroup Driver `systemd`** với **cgroup v2**: Docker giao việc quản lý cgroup cho systemd, nên mỗi container là một scope nằm trong cây cgroup chung của hệ thống. Đây là cấu hình được khuyến nghị vì chỉ có một trình quản lý cgroup duy nhất, tránh xung đột khi tài nguyên bị thiếu.

## 5. Thông số trích xuất từ `docker info`

Lệnh chạy bằng user `devops` **không cần `sudo`** (user đã thuộc group `docker`).

```bash
docker version
docker info
docker info | grep -iE "server version|cgroup|containers|running|paused|stopped|images"
```

```text
 Containers: 3
  Running: 1
  Paused: 0
  Stopped: 2
 Images: 4
 Server Version: 29.9.0
 Cgroup Driver: systemd
 Cgroup Version: 2
  cgroupns
```

| Thông số | Giá trị | Ý nghĩa |
|---|---|---|
| Server Version | `29.9.0` | Phiên bản Docker Engine (daemon), trùng với Client |
| Cgroup Driver | `systemd` | systemd quản lý cgroup cho container |
| Cgroup Version | `2` | Dùng cgroup v2 (unified hierarchy); `Security Options` có `cgroupns`, tức mỗi container còn có cgroup namespace riêng |
| Containers | `3` (Running `1`, Paused `0`, Stopped `2`) | Tổng container, gồm cả đang chạy và đã dừng; container đang chạy là `html-app` của bài trước |
| Images | `4` | Số image đang lưu trên máy (`hello-world`, `nginx:alpine`, `my-html-app:v1`, `alpine`) |
| Storage Driver | `overlayfs` | Ghép các layer chỉ đọc của image và lớp ghi được của container |
| Kernel / OS | `5.15.0-198-generic` / Ubuntu 22.04.5 LTS | Kernel dùng chung cho mọi container |
| Tài nguyên host | 1 CPU, 1.918 GiB RAM | Giới hạn trên cho tổng tài nguyên các container |

![docker version](images/01-docker-version.png)

![docker info](images/02-docker-info.png)

![docker info - thông số chính](images/03-docker-info-grep.png)

## 6. Kết luận và khuyến nghị

1. **Tiết kiệm tài nguyên:** container bỏ đi Guest OS nên trên cùng một VPS có thể chạy nhiều ứng dụng thay vì thuê thêm VPS cho từng ứng dụng.
2. **Triển khai nhanh và đồng nhất:** một image chạy giống hệt nhau ở mọi môi trường, khởi động dưới 1 giây, giải quyết lỗi "chạy được ở máy dev nhưng lỗi trên server".
3. **Vẫn đảm bảo an toàn và ổn định:** Namespaces cô lập Process, Network, Mount; cgroups giới hạn CPU, RAM để một ứng dụng lỗi không kéo sập cả máy.
4. **Lưu ý:** vì dùng chung kernel, cần cập nhật kernel host thường xuyên, chạy ứng dụng bằng user không phải root trong container và chỉ cấp quyền group `docker` cho người tin cậy (quyền này tương đương root).

**Khuyến nghị:** giữ VPS làm nền hạ tầng, chuyển các ứng dụng sang chạy dạng container trên đó, quản lý bằng Docker Compose và luôn đặt giới hạn `--cpus` / `--memory` cho từng service.
