---
title: "Giới thiệu AWS Lambda"
date: "`r Sys.Date()`"
weight: 1
chapter: false
---

# Giới thiệu AWS Lambda

#### Tổng quan

Trong bài lab này, bạn sẽ tìm hiểu về dịch vụ **AWS Lambda**, một trong những dịch vụ quan trọng nhất trong hệ sinh thái AWS giúp bạn chạy mã mà không cần quản lý máy chủ. Bạn sẽ nắm được khái niệm Lambda là gì, khi nào nên sử dụng và cách Lambda được kích hoạt (trigger). Ngoài ra, phần này cũng cung cấp tổng quan về cách **AWS Lambda tính phí** – điều rất quan trọng khi bạn triển khai thực tế.

---

#### AWS Lambda là gì?

**AWS Lambda** là một dịch vụ **serverless computing** do Amazon Web Services cung cấp. Bạn chỉ cần **viết code**, chọn ngôn ngữ (Python, Node.js, Java, v.v.), triển khai và AWS sẽ tự động:
- Cấp phát tài nguyên (compute)
- Quản lý hệ điều hành, bảo mật
- Scale tự động
- Theo dõi và log hoạt động

Lambda phù hợp cho các tác vụ nhỏ, đơn lẻ, không cần máy chủ chạy 24/7.

![AWS Lambda Service Overview](/images/lambda/lambda-overview.png?featherlight=false&width=90pc)

{{% notice info %}}
Lambda là một dịch vụ dạng **event-driven**, nghĩa là chỉ khi có sự kiện xảy ra (trigger), mã của bạn mới chạy.
{{% /notice %}}

---

#### Khi nào nên dùng Lambda?

Bạn nên cân nhắc sử dụng AWS Lambda trong các tình huống:

| Tình huống                       | Lý do phù hợp                                |
|----------------------------------|-----------------------------------------------|
| Xử lý ảnh sau khi upload lên S3  | Trigger từ S3, xử lý ảnh rồi lưu lại          |
| Gửi email tự động khi có sự kiện | Kết hợp với SNS, SES, hoặc EventBridge        |
| Backend nhỏ cho REST API         | Tích hợp với API Gateway                      |
| Định kỳ kiểm tra/tự động hóa     | Tạo cron job bằng EventBridge                 |
| Nhận và xử lý tin nhắn           | Từ SQS, SNS, DynamoDB Streams hoặc Kinesis    |

---

#### Cơ chế Trigger của Lambda

Một Lambda function sẽ chỉ thực thi khi có **trigger** xảy ra từ các dịch vụ khác hoặc bạn gọi trực tiếp nó.

##### 🔄 Một số nguồn trigger phổ biến:
- **Amazon S3**: khi có object được upload/xóa
- **API Gateway**: khi người dùng gọi REST API
- **DynamoDB Streams**: khi có dữ liệu được ghi vào bảng
- **CloudWatch Events / EventBridge**: kích hoạt định kỳ
- **SNS / SQS**: khi có tin nhắn mới

{{% notice tip %}}
Trigger giúp bạn xây dựng hệ thống **phản ứng tự động** mà không cần polling hay cron server phức tạp.
{{% /notice %}}

---

#### Cách tính chi phí (Pricing)

Lambda tính chi phí theo **2 yếu tố chính**:
1. **Số lần gọi function**
2. **Thời gian chạy và bộ nhớ cấp phát**

##### 🎯 Công thức tính:
```text
Giá = (thời gian chạy tính theo mili giây) × (bộ nhớ) × (số lượt gọi)
