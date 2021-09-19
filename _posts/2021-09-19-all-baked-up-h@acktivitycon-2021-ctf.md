---
title: All Baked Up - H@acktivityCon 2021 CTF
author: HoangND
date: 2021-09-19 19:00:00 +0700
categories: [Writeups, H@acktivityCon 2021 CTF]
tags: [graphql, web, sqli]
---

All Baked Up là một trang web được dùng để lưu giữ các công thức làm bánh. Chức năng chỉ dừng lại ở trang liệt kê danh sách và truy cập cụ thể vào từng lại bánh thôi.

![image](https://user-images.githubusercontent.com/61985236/133925541-93711063-56f1-4c48-b1f3-5e0f20b560de.png)

Phânn tích các request của nó

![image](https://user-images.githubusercontent.com/61985236/133925563-52393185-11ed-428f-b5a7-61f1220ebd08.png)

Vậy là trang web sử dụng công nghệ graphql để thực hiện truy vấn. Vậy chính xác graphql là gì. Mình viết luôn vào blog này vì mình cũng mới tìm hiểu về nó thôi 😌.

GraphQL là ngôn ngữ truy vấn cho các API và runtime để thực hiện các truy vấn đó với dữ liệu. Nghĩa là người dùng sẽ tương tác với API thông qua graphql

![image](https://user-images.githubusercontent.com/61985236/133925714-1ee62ce8-a70a-4019-a78f-4c3ecdae1b7f.png)

Mục đích của graphql là tạo ra một endpoint đơn giản dễ hiểu dễ sử dụng để handle tất cả các endpoint phức tạp còn lại. 
