# ترفندهای Kubernetes و DevOps

مجموعه‌ای از نکته‌ها و دستورهای کاربردی روزمره. هر بخش مستقل است.

## ۱. kubectl

### alias و autocomplete
```bash
alias k=kubectl
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

### جابه‌جایی سریع namespace
```bash
kubectl config set-context --current --namespace=my-ns
```

### تولید YAML بدون ساخت منبع
```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
kubectl run tmp --image=busybox --dry-run=client -o yaml -- sleep 3600
```

### بررسی تغییرات قبل از apply
```bash
kubectl diff -f web.yaml
kubectl apply -f web.yaml --server-side --dry-run=server
```

### پیدا کردن podهای مشکل‌دار
```bash
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get pods -A | grep -Ev 'Running|Completed'
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

### مصرف منابع
```bash
kubectl top pods -A --sort-by=memory | head
kubectl top nodes
```

### خروجی سفارشی با jsonpath / custom-columns
```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
```

### لاگ چند pod با هم
```bash
kubectl logs -l app=web --all-containers --tail=50 --prefix -f
kubectl logs my-pod --previous        # لاگ کانتینرِ قبل از restart
```

### دیباگ pod بدون shell (ephemeral container)
```bash
kubectl debug -it my-pod --image=nicolaka/netshoot --target=app
kubectl debug node/my-node -it --image=ubuntu    # دسترسی به نود
```

### rollout
```bash
kubectl rollout status deploy/web
kubectl rollout undo deploy/web
kubectl rollout restart deploy/web
```

### ایجاد port-forward سریع
```bash
kubectl port-forward svc/web 8080:80
```

### بررسی دسترسی RBAC
```bash
kubectl auth can-i create pods --as=system:serviceaccount:dev:ci
kubectl auth can-i --list -n dev
```

## ۲. عیب‌یابی خطاهای رایج

| وضعیت | علت محتمل | بررسی |
|---|---|---|
| `ImagePullBackOff` | نام/تگ اشتباه یا نبودن imagePullSecret | `kubectl describe pod` بخش Events |
| `CrashLoopBackOff` | خطای برنامه یا probe اشتباه | `kubectl logs --previous` |
| `Pending` | کمبود منابع، taint، PVC نچسبیده | `kubectl describe pod` و `kubectl get pvc` |
| `OOMKilled` | limit حافظه کم است | `kubectl describe pod` → Last State |
| `Evicted` | فشار روی دیسک/حافظه نود | `kubectl describe node` |
| سرویس جواب نمی‌دهد | selector نمی‌خواند | `kubectl get endpoints svc-name` |

نکته: اگر `endpoints` خالی بود، سرویس هیچ podی پیدا نکرده؛ label و selector را مقایسه کن.

## ۳. مدیریت منابع و پایداری

- همیشه `requests` و `limits` بگذار. برای CPU معمولاً limit را سخت‌گیرانه نگذار تا throttling نگیری.
- برای هر Deployment حیاتی یک `PodDisruptionBudget` تعریف کن.
- `readinessProbe` و `livenessProbe` را جدا از هم و با منطق متفاوت بنویس. liveness نباید به وابستگی‌های بیرونی (مثل دیتابیس) وابسته باشد.
- برای پخش بار بین نودها از `topologySpreadConstraints` استفاده کن.
- از `startupProbe` برای برنامه‌های کند در بالا آمدن استفاده کن.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: web
```

## ۴. امنیت

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
```

- Secretها در etcd فقط base64 هستند. از Sealed Secrets، External Secrets یا SOPS استفاده کن و Secret خام را در Git نگذار.
- برای ServiceAccountها حداقل دسترسی بده و `automountServiceAccountToken: false` را در جاهای غیرضروری بگذار.
- با NetworkPolicy پیش‌فرض را deny کن و فقط مسیرهای لازم را باز کن.
- imageها را به digest یا تگ ثابت pin کن، نه `latest`.

## ۵. Helm

```bash
helm template my-release ./chart -f values.yaml     # رندر بدون نصب
helm lint ./chart
helm upgrade --install app ./chart -n prod --create-namespace --atomic --timeout 5m
helm diff upgrade app ./chart -f values.yaml        # نیاز به افزونه helm-diff
helm get values app -n prod
helm history app -n prod && helm rollback app 3 -n prod
```

- `--atomic` در صورت شکست، خودکار rollback می‌کند.
- برای اینکه با تغییر ConfigMap، podها restart شوند، checksum را در annotation بگذار:
```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

## ۶. Docker

```bash
docker system df                         # مصرف دیسک
docker system prune -af --volumes        # احتیاط: همه‌چیزِ بلااستفاده را پاک می‌کند
docker run --rm -it --entrypoint sh image:tag
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.0 --push .
```

نکته‌های Dockerfile:
- از multi-stage build استفاده کن تا image نهایی کوچک شود.
- لایه‌هایی که کمتر تغییر می‌کنند (نصب dependency) را بالاتر بگذار تا cache کارا باشد.
- فایل `.dockerignore` را حتماً داشته باش.
- کاربر غیر root تعریف کن.

## ۷. CI/CD و GitOps

- هر چیزی که در کلاستر هست باید در Git باشد. دستی `kubectl edit` نکن؛ ابزارهایی مثل ArgoCD یا Flux تغییر دستی را برمی‌گردانند.
- یک تگ immutable برای هر build بساز (مثلاً hash کامیت) و در محیط‌ها promote کن، نه rebuild.
- در pipeline مراحل lint، test، scan امنیتی image (مثل Trivy) و بعد deploy را بگذار.

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 repo/app:1.0
kubectl apply --dry-run=server -f manifests/
kubeconform -strict manifests/
```

## ۸. مانیتورینگ و لاگ

- چهار سیگنال طلایی: latency، traffic، errors، saturation.
- برای هشدار روی symptom هشدار بده (خطای کاربر)، نه روی هر علتِ ممکن.
- مثال PromQL برای نرخ خطا:
```promql
sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```
- برای کاهش هزینه‌ی cardinality از label با مقدار نامحدود (مثل user-id) در متریک‌ها پرهیز کن.

## ۹. لینوکس و شبکه (روی نود)

```bash
journalctl -u kubelet -f
crictl ps && crictl logs <id>
ss -tulpn
dig +short svc-name.ns.svc.cluster.local     # از داخل pod
iptables -t nat -L -n | grep <service-ip>
```

## ۱۰. عادت‌های خوب

- قبل از تغییر بزرگ، `kubectl diff` و محیط staging.
- برچسب‌گذاری یکسان: `app.kubernetes.io/name`, `app.kubernetes.io/version`, `app.kubernetes.io/part-of`.
- منابع را با `ResourceQuota` و `LimitRange` در هر namespace محدود کن.
- از backup دوره‌ای etcd و تست restore غافل نشو.
- پیش از ارتقای نسخه‌ی کلاستر، API های deprecated را با ابزاری مثل `pluto` یا `kubent` بررسی کن.

---
لایسنس پیشنهادی برای این ریپو: MIT (نام خودت را در فایل LICENSE بگذار).
