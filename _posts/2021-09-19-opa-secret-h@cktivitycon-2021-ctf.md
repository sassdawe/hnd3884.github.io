---
title: OPA Secret - H@acktivityCon 2021 CTF
author: HoangND
date: 2021-09-19 15:55:00 +0700
categories: [Writeups, H@cktivityCon 2021 CTF]
tags: [ssrf, web]
---

OPA secret là một trang web cho phép bạn quản lý các khóa bí mật của mình, có chức năng đăng ký đăng nhập. Giao diện quản lý các khóa như sau

![image](https://user-images.githubusercontent.com/61985236/133922403-e86d93ac-7701-4862-8fbf-2e57904bc83f.png)

Ở đây có các chức năng tạo khóa, thay đổi giá trị khóa và phân quyền read và write

![image](https://user-images.githubusercontent.com/61985236/133922416-c815a835-c824-45b5-8f4a-2547bb225c53.png)

Ở phần setting tài khoản, có thể đặt ảnh đại diện cho tài khoản bằng url dẫn tới ảnh, cái này có tiềm năng cho tấn công ssrf.

![image](https://user-images.githubusercontent.com/61985236/133922495-6b2faa6b-c38d-45e1-98d8-d6d5ff4d5b8c.png)

Oke vậy là đã đi qua các chức năng của trang web, giờ cần ngó sang phần mã nguồn, toàn bộ phần xử lý backend được thực hiện bên trong app.py. Sau một hồi xem xét 
mình thấy các hàm xử lý có gọi đến một internal url

![image](https://user-images.githubusercontent.com/61985236/133922618-5caa426c-880f-4b62-a8ac-174667d5fbd5.png)

khả năng cao là web còn thay đổi dữ liệu của database chứ không chỉ thao tác với dữ liệu qua các biến mảng. Kết hợp điều này với chức năng thay đổi ảnh đại diện kết quả sẽ là SSRF.

Tiếp theo là xem xét flag nằm ở đâu. Trang web này khi vừa chạy sẽ tự động tạo ra một số account và khóa ban đầu. Trong đó khóa flag thuộc về user admin

![image](https://user-images.githubusercontent.com/61985236/133922963-6b13cad3-ffbd-45cf-be13-b6345195b759.png)

Oke vậy để lấy được cờ thì mình phải có trong tay một tài khoản user có quyền ít nhất là read cái khóa này. Kiểm tra chức năng phân quyền khóa

![image](https://user-images.githubusercontent.com/61985236/133923113-1af79b31-dd7c-4427-81e2-ef4b085f7855.png)

Hàm add_reader thực thi công việc thêm quyền read khóa cho một user theo id của user đó, để đỡ mất công mình sẽ lấy luôn id của một trong các tài khoản được tạo ban đầu khỏi cần tạo tài khoản khác

```
{"id":"243eae36-621a-47a6-b306-841bbffbcac4"}
```

và id khóa flag như sau

```
{"id": "afce78a8-23d6-4f07-81f2-47c96ddb10cf"}
```

Oke vậy là đã có request để thêm quyền đọc khóa flag cho user của mình. Vì là ssrf nên không cần xác thực gì thêm. Tiếp theo xem đoạn code xử lý thay ảnh đại diện, như đã nói ở trên đây là vị trí tiềm răng cho ssrf.

![image](https://user-images.githubusercontent.com/61985236/133923449-bc9a729e-3f1f-4d7b-b5b4-0b95a82c9734.png)

Url ảnh được đưa qua blacklist, sau đó được gọi bằng os command là curl

```python
cmd = f"curl --request GET {url} --output ./static/images/{user['id']} --proto =http,https"
status = os.system(cmd)
```

Ở đây command bắt buộc là get, tuy nhiên request mình cần là put. Như vậy mình cần tạo thêm một curl request nữa thông qua url. Ở đây có một kỹ thuật sử dụng %0a là url encode của ký tự xuống hàng.
Khi này command được thực thi bởi os sẽ bị tách ra làm hai command riêng. Sau khi thử nghiệm thì url đầy đủ của mình như sau

```
url=https://i.pinimg.com/originals/d0/e0/60/d0e06034b750ed272354454ec3ef9dc2.jpg%0acurl%20--request%20PUT%20--header%20"Content-Type:%20application/json"%20--data%20[b2f0aa09-4a78-4bfd-9367-8f0e77d05efd]%20http://localhost:8181/v1/data/readers/afce78a8-23d6-4f07-81f2-47c96ddb10cf
```
Kết quả gửi request

![image](https://user-images.githubusercontent.com/61985236/133923826-d693ecc1-ec0d-47f0-926a-d8be1b01d803.png)

Vậy là đã thêm quyền đọc flag thành công, bây giờ chỉ cần gọi request để lấy value của khóa thôi

![image](https://user-images.githubusercontent.com/61985236/133923842-66d271b6-59a5-4128-930e-f9f5f6998efd.png)
