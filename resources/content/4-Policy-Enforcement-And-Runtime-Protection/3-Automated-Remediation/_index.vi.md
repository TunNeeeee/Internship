---

title: "Tự động phản ứng với vi phạm bằng Falcosidekick"
date: 2025-07-15
weight: 3
chapter: false
pre: " 4.3 "

---

## Nội dung chính

* [Giới thiệu Falcosidekick](#giới-thiệu-falcosidekick)
* [Triển khai Falcosidekick cùng Falco](#triển-khai-falcosidekick-cùng-falco)
* [Cấu hình gửi cảnh báo](#cấu-hình-gửi-cảnh-báo)
* [Kiểm thử cảnh báo](#kiểm-thử-cảnh-báo)

---

## Giới thiệu Falcosidekick

**Falcosidekick** là một dịch vụ hỗ trợ Falco gửi cảnh báo đến các hệ thống bên ngoài như:

* Slack, Discord
* Email
* Webhook, Kafka, NATS
* Prometheus, Grafana, Elasticsearch...

Falcosidekick nhận log từ Falco và chuyển tiếp theo cấu hình.

---

## Triển khai Falcosidekick cùng Falco

### Bước 1: Cài Falco kèm Falcosidekick

```bash
helm install falcosidekick falcosecurity/falcosidekick --namespace falco --set config.slack.webhookurl="https://hooks.slack.com/services/xxx"
```

🔁 Có thể thay Slack bằng webhook URL khác tùy nhu cầu.

### Bước 2: Kiểm tra pod đã chạy

```bash
kubectl get pods -n falco
```

✅ Kết quả mong đợi: Có 2 pod đang chạy: `falco` và `falcosidekick`

---

## Cấu hình gửi cảnh báo

Bạn có thể chỉnh sửa thêm trong file `values.yaml` nếu muốn cấu hình nhiều hệ thống cảnh báo.

Ví dụ: Gửi về Discord, Prometheus, Webhook...

```yaml
falcosidekick:
  config:
    discord:
      webhookurl: "https://discord.com/api/webhooks/xxx"
    webhook:
      address: "http://your-api-endpoint"
```

Sau đó nâng cấp chart:

```bash
helm upgrade falco falcosecurity/falco -n falco -f values.yaml
```

---

## Kiểm thử cảnh báo

### Gây sự kiện bất thường

```bash
kubectl run -i --tty attacker --image=alpine -- sh
```

Sau đó chạy:

```bash
touch /etc/passwd
```

### Kiểm tra alert gửi đi

* Kiểm tra log Falcosidekick:

```bash
kubectl logs -l app=falcosidekick -n falco
```

* Xem thông báo trên Slack/Discord/Webhook tùy theo cấu hình.

📸 *Chụp ảnh minh họa alert nhận được ở Slack hoặc Discord để đưa vào tài liệu.*

---

## Ghi chú

* Falcosidekick giúp tích hợp Falco với nhiều hệ thống alerting khác nhau.
* Có thể mở rộng gửi alert đến SIEM, hệ thống automation (XDR, SOAR).


