# ApplicationSet — cách hoạt động, khi nào dùng, và troubleshooting

Ghi lại từ lần triển khai thật trên cluster này. Các case ở cuối là lỗi thật
đã gặp và đã sửa, không phải ví dụ giả định.

## Vấn đề nó giải quyết

Một `Application` = một app trên một cluster. Muốn thêm môi trường thì phải
viết thêm một `Application`, và từ đó phải nhớ đồng bộ tay mọi thay đổi giữa
các file gần-giống-nhau. Đây là nguồn drift kinh điển: sửa `retry.limit` ở
prod, quên staging, ba tháng sau mới phát hiện.

`ApplicationSet` là **template + generator**: generator sinh danh sách tham
số, template biến mỗi phần tử thành một `Application`. Repo này dùng git
generator quét `envs/*/values.yaml`, nên thêm môi trường = thêm một thư mục.
Không có `Application` mới phải viết.

## Khi nào NÊN dùng

- **Nhiều môi trường cùng một chart** — dev/staging/uat/prod.
- **Nhiều cluster** — cùng app đẩy ra nhiều region, dùng cluster generator.
- **Preview environment cho mỗi PR** — pull request generator, tự sinh khi mở
  PR và tự xoá khi đóng.
- **Monorepo nhiều service** — mỗi thư mục con là một app.

## Khi nào KHÔNG nên

- **Chỉ một app, một môi trường.** Thêm một lớp trừu tượng để sinh đúng một
  `Application` chỉ làm khó debug.
- **Các môi trường khác nhau về bản chất**, không chỉ khác giá trị. Khi
  template phải nhồi đầy điều kiện thì hai `Application` riêng dễ đọc hơn.
- **Cần kiểm soát từng app độc lập.** ApplicationSet sẽ reconcile lại mọi
  thay đổi tay trên Application nó quản lý.

## Các generator phổ biến

| Generator | Sinh ra từ | Dùng khi |
| --- | --- | --- |
| **list** | mảng cố định trong YAML | vài môi trường ổn định, muốn tường minh |
| **git (files)** | file khớp glob | mỗi env một values file — **repo này dùng** |
| **git (directories)** | thư mục khớp glob | mỗi thư mục là một app, không cần đọc nội dung |
| **cluster** | cluster đã đăng ký trong ArgoCD | fan-out cùng app ra nhiều cluster |
| **pull request** | PR đang mở trên SCM | preview env, tự tạo và tự xoá |
| **matrix** | tích Descartes của 2 generator | ví dụ (mỗi cluster) × (mỗi env) |
| **merge** | gộp nhiều generator, ưu tiên theo key | lấy list làm nền rồi override |
| **scm provider** | mọi repo trong org | onboard hàng loạt repo |

`matrix` hay bị hỏi khi phỏng vấn: nó nhân hai generator với nhau, nên
2 cluster × 3 env = 6 Application. Rất mạnh, và cũng rất dễ vô tình sinh ra
hàng chục app.

## RollingSync

```yaml
strategy:
  type: RollingSync
  rollingSync:
    steps:
      - matchExpressions: [{key: env, operator: In, values: [staging]}]
      - matchExpressions: [{key: env, operator: In, values: [prod]}]
```

Không có nó, một commit sửa chart sẽ đẩy đồng thời vào mọi môi trường — mất
luôn ý nghĩa của việc có staging. RollingSync bắt staging phải Healthy trước
khi prod bắt đầu.

**Lưu ý:** `matchExpressions` khớp trên **label của Application**, nên template
bắt buộc phải gắn label đó (`labels: {env: "{{.path.basename}}"}`). Thiếu label
thì step không khớp app nào và RollingSync im lặng không làm gì cả.

---

# Troubleshooting — case thật đã gặp

## Case 1: Application tên `todolist-{{.path.basename}}`

**Hiện tượng:** ApplicationSet tạo xong nhưng không sinh app nào.

```
ErrorOccurred: True | Application.argoproj.io "todolist-{{.path.basename}}"
is invalid: metadata.name: Invalid value ... RFC 1123 subdomain
ParametersGenerated: True        <-- generator CHẠY ĐÚNG
```

**Nguyên nhân:** thiếu `goTemplate: true`. ArgoCD có hai engine template:

| Engine | Syntax | Bật bằng |
| --- | --- | --- |
| fasttemplate (legacy) | `{{path.basename}}` — không dấu chấm | mặc định |
| Go text/template | `{{.path.basename}}` — **có** dấu chấm | `goTemplate: true` |

Viết syntax mới mà không bật engine mới thì chuỗi giữ nguyên nguyên văn, và
API server từ chối vì tên không hợp RFC 1123.

**Chẩn đoán nhanh:** `ParametersGenerated: True` nhưng tên app còn `{{ }}` thì
gần như luôn là lỗi engine template, không phải lỗi generator.

**Fix:**
```yaml
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]   # báo lỗi thay vì in "<no value>"
```

## Case 2: Generator chỉ thấy 1 môi trường dù local có 2

**Hiện tượng:** `envs/staging/` tồn tại trên máy, log vẫn `generated 1 applications`.

**Nguyên nhân:** git generator đọc từ **remote repo**, không đọc đĩa local.
Thư mục chưa commit và push thì với ArgoCD nó không tồn tại.

**Bài học:** với GitOps, "đã tạo file" không có nghĩa gì cho tới khi push.
Kiểm tra bằng `git show origin/main:<path>`, đừng đọc file local.

## Case 3: Sync treo — deadlock kinh điển

**Hiện tượng:** staging `OutOfSync/Progressing` mãi. Đã push fix, đã hard
refresh, đã xoá Rollout — vẫn thấy image cũ.

```
operationState.phase: Running   started 18:11:49    <-- treo ~15 phút
operationState.operation.sync.revision: d300155     <-- commit CŨ
message: waiting for healthy state of argoproj.io/Rollout/todolist-frontend
```

**Nguyên nhân:** deadlock. ArgoCD chờ Rollout Healthy mới hoàn tất sync;
Rollout không bao giờ Healthy vì đang chạy image lỗi; commit sửa lỗi xếp hàng
sau operation đang treo. **Xoá Rollout không cứu được** — ArgoCD tạo lại từ
manifest của operation cũ, tức là lại image cũ.

Dấu hiệu kèm theo: log repo-server hiện `manifest cache hit` với `cacheKey`
chứa commit cũ.

**Fix — phải huỷ operation trước, mọi cách khác vô nghĩa:**
```bash
kubectl patch app <app> -n argocd --type merge -p '{"operation":null}'
kubectl patch app <app> -n argocd --type json \
  -p '[{"op":"remove","path":"/status/operationState"}]'
kubectl annotate app <app> -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

Hoặc `argocd app terminate-op <app>` nếu có CLI.

**Bài học phỏng vấn:** khi ArgoCD "không chịu nhận commit mới", kiểm tra
`operationState.phase` TRƯỚC TIÊN. Còn `Running` là còn giữ lock; hard refresh
và xoá resource đều không phá được lock đó.

## Case 4: `:latest` làm sập môi trường mới

**Hiện tượng:** staging frontend `0/1`, restart liên tục. nginx khởi động bình
thường nhưng probe báo `connection refused` ở 8080.

**Nguyên nhân:** CI chỉ ghi digest vào `envs/prod/values.yaml`, staging còn
`:latest`. Tag đó trỏ tới image **trước** khi đổi sang nginx-unprivileged —
image cũ listen cổng 80, chart mới probe 8080.

**Nguyên nhân gốc:** tag mutable làm một image không còn khớp chart sống lại
âm thầm. Đây đúng là lý do tồn tại của việc pin digest.

**Fix:** cho CI ghi vào **mọi** overlay:
```yaml
GITOPS_VALUES: "envs/staging/values.yaml envs/prod/values.yaml"
```

**Bài học:** thêm môi trường thì phải kiểm tra cả đường ghi của CI. Thiếu một
overlay là thiếu âm thầm — không báo lỗi, môi trường đó chỉ đóng băng ở image
cũ cho tới khi nó vỡ.

## Case 5: Argo Rollouts copy ingress làm API trả 404

**Hiện tượng:** `/api` trả 404 qua host công khai, nhưng gọi trực tiếp
ClusterIP Service thì 200.

**Nguyên nhân:** Argo Rollouts tạo canary ingress bằng cách **copy toàn bộ**
stable ingress. Ingress đó có `/api`, nên canary cũng claim `/api` nhưng
backend lại là Service **frontend** canary. Ở weight 0% vẫn tồn tại hai claim
cho cùng host và path, và nginx trả 404 cho API.

**Fix:** ingress mà Rollouts quản lý chỉ chứa path của frontend. API tách sang
một ingress riêng mà Rollouts không chạm tới.

**Bẫy kèm theo:** trong lúc chuyển đổi, nginx log
`host ... and path "/api" is already defined in ingress todolist/todolist`.
Khi nginx từ chối một config, nó từ chối **cả host** nên mọi path đều 404 —
không riêng path xung đột. Đừng chẩn đoán sai thành "app đã chết".

## Case 6: 404 nhưng nguyên nhân nằm ngoài Kubernetes

**Hiện tượng:** NodePort trả 200, host công khai trả 404.

**Nguyên nhân:** cluster này nằm sau một Traefik container sở hữu cổng 80/443
cho các app khác. Có người sửa `/opt/traefik/dynamic.yml` để thêm router mới
và làm mất router trỏ vào k3s.

**Bài học:** cô lập từng tầng trước khi sửa. `curl` vào NodePort trả 200 nghĩa
là ingress và app đều đúng, vấn đề ở tầng edge phía trước. Lần đó tôi đã sửa
đúng một bug ingress trong Kubernetes, nhưng 404 lúc ấy đến từ **hai nguyên
nhân độc lập** — sửa một cái không làm hết triệu chứng.

---

## Checklist debug ApplicationSet

```bash
# 1. Generator có sinh tham số?
kubectl get applicationset <name> -n argocd \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} | {.message}{"\n"}{end}'

# 2. Sinh ra bao nhiêu app, có lỗi khi tạo không?
kubectl logs -n argocd deploy/argocd-applicationset-controller --tail=50 \
  | grep -iE 'generated|error'

# 3. App có bị treo operation? (kiểm tra TRƯỚC khi thử bất cứ thứ gì khác)
kubectl get app <app> -n argocd \
  -o jsonpath='{.status.operationState.phase} {.status.operationState.operation.sync.revision}{"\n"}'

# 4. Manifest render đúng như mong đợi? (so với remote, không phải local)
git show origin/main:envs/<env>/values.yaml
helm template <rel> charts/todolist -f envs/<env>/values.yaml | grep 'image:'

# 5. RollingSync không chạy? Kiểm tra label trên Application
kubectl get app -n argocd --show-labels
```
