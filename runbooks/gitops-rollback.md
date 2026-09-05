# GitOps Drift & Rollback Runbook — Lab 28

Tài liệu quy trình và bằng chứng kiểm thử triển khai Kubernetes Manifests, quản lý cấu hình tập trung bằng GitOps (Argo CD), cơ chế tự phục hồi (Self-Healing) và quy trình Rollback phiên bản theo tiêu chuẩn `runbooks/gitops-rollback.md` và `SUBMISSION.md`.

---

## 1. Xác thực Hợp đồng Kubernetes & GitOps Manifests

### 1.1. Thực thi Kiểm tra Tự động (Contract Validation)
Chạy kịch bản kiểm định tĩnh cấu hình Kubernetes trong thư mục `deploy/kubernetes/base` và `gitops/`:
```bash
uv run python scripts/validate_manifests.py
```
**Kết quả thực tế:**
```text
Kubernetes and GitOps manifest contracts passed
Exit code: 0
```

### 1.2. Các Hợp đồng Nghiêm ngặt Đã Kiểm chứng
1. **Đầy đủ 9 Kubernetes Kinds bắt buộc:**
   - `Deployment`, `Service`, `ServiceAccount`, `ConfigMap`, `HorizontalPodAutoscaler`, `PodDisruptionBudget`, `NetworkPolicy`, `Gateway`, `HTTPRoute`.
2. **Tiêu chuẩn An toàn Bảo mật Container (Pod Security Standards):**
   - Không chạy với quyền root (`runAsNonRoot: true`).
   - Image không sử dụng tag `:latest` (bắt buộc ghim phiên bản tag cố định, ví dụ `v3.0.0`).
   - Khai báo đầy đủ `readinessProbe`, `livenessProbe`, `resources` (requests/limits) và `securityContext`.
3. **Gateway API Chuẩn hóa:**
   - Sử dụng chuẩn `gateway.networking.k8s.io/v1` (thay vì Ingress cũ đã bị deprecated).
4. **GitOps Desired State Pinning:**
   - Tệp `gitops/application.yaml` ghim `targetRevision` vào release tag (`refs/tags/v3.0.0`), nghiêm cấm trỏ vào nhánh động `main`/`master`/`HEAD`.

---

## 2. Kịch bản Mô phỏng Configuration Drift & Cơ chế Tự phục hồi (Self-Healing)

### 2.1. Cấu hình Tự phục hồi trong Argo CD (`gitops/application.yaml`)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lab28-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/VinUni-AI20k/Day28-Modern-Platform-Lab-Student.git
    targetRevision: refs/tags/v3.0.0
    path: deploy/kubernetes/base
  destination:
    server: https://kubernetes.default.svc
    namespace: lab28
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  revisionHistoryLimit: 5
```

### 2.2. Kịch bản Diễn tập Trôi dạt Cấu hình (Drift Injection)
1. **Trạng thái chuẩn (Desired State):**
   - Deployment `lab28-api` có `replicas: 2` theo Kustomize overlay.
2. **Bơm sai lệch trực tiếp trên cụm K8s (Drift Trigger):**
   - Kỹ sư can thiệp thủ công bằng lệnh ad-hoc:
     ```bash
     kubectl scale deployment lab28-api --replicas=5 -n lab28
     ```
3. **Quan sát phản ứng của GitOps Controller:**
   - Argo CD phát hiện sự sai lệch giữa Live State (`replicas: 5`) và Git Desired State (`replicas: 2`).
   - Trạng thái ứng dụng chuyển sang `OutOfSync (Drift Detected)`.
   - Do cờ `syncPolicy.automated.selfHeal: true` được kích hoạt, Argo CD Reconciliation Loop tự động đảo ngược thay đổi thủ công, scale số lượng pod trở lại đúng **2 replicas**.
   - Trạng thái quay trở lại `Synced` & `Healthy`.

---

## 3. Quy trình Rollback Phiên bản (Zero-Downtime Rollback Procedure)

Khi phát hiện phiên bản mới (`v3.1.0`) gặp lỗi logic serving, thực hiện rollback về phiên bản an toàn (`v3.0.0`) theo đúng triết lý GitOps:

### Bước 1: Kiểm tra Lịch sử Triển khai (History Check)
```bash
argocd app history lab28-platform
```
Xác định ID của phiên bản ổn định trước đó (ví dụ Revision ID tương ứng với tag `v3.0.0`).

### Bước 2: Thực hiện Rollback Desired State qua Git
Trong GitOps thật, cách thức rollback chuẩn mực nhất là revert commit trên Git repository để lịch sử thay đổi luôn minh bạch:
```bash
git revert <bad-commit-hash>
git push origin main
```
Hoặc trong tình huống khẩn cấp qua Argo CD CLI:
```bash
argocd app rollback lab28-platform <revision-id>
```

### Bước 3: Xác minh Hệ thống sau Rollback
1. **Replicas & Pod Health:**
   - Kubernetes thực hiện Rolling Update ngược lại, các pod phiên bản cũ dần terminate trong khi pod phiên bản ổn định khởi chạy.
   - `PodDisruptionBudget` đảm bảo luôn có ít nhất 1 pod sẵn sàng phục vụ.
2. **API Gateway Routing:**
   - Envoy Gateway tự động cập nhật endpoint clusters, không có bất kỳ request nào bị drop 502 Bad Gateway.
3. **Trace Continuity:**
   - Các request gửi tới Gateway vẫn mang header `traceparent` đồng nhất, lan truyền span `lab28.gateway.request` sang pod của phiên bản đã rollback.
