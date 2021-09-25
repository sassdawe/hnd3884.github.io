---
title: Chainreaction - DownUnderCTF 2021
author: HoangND
date: 2021-09-25 15:55:00 +0700
categories: [Writeups, DownUnderCTF 2021]
tags: [xss, web, NFKD normalised]
---

![image](https://user-images.githubusercontent.com/61985236/134768470-0e313bc6-67b4-487a-9542-1ebcc067b6ce.png)

Trang chủ của Chainreaction không có gì đáng bận tâm, ngoại trừ việc ảnh không load được, nhưng hiện tại vẫn không biết lí do thôi bỏ qua. Có hai button đăng ký đăng nhập. 
Ở đây các form này không khai thác được SQLi. Quan sát một chút form đăng ký

![image](https://user-images.githubusercontent.com/61985236/134768675-1fec4bb8-fc48-4249-b28e-fa60a39efc4a.png)

Username không được chứa các ký tự < và >. Vậy khả năng cao có thể bypass để khai thác lỗi XSS qua username. Cũng có một điều đặc biệt bên dưới form đăng nhập làm mình chú ý

![image](https://user-images.githubusercontent.com/61985236/134768537-778e468c-5013-4c07-968d-13b121315325.png)

vậy là có trang đăng nhập riêng cho deverloper, vào luôn xem có gì hay ho

![image](https://user-images.githubusercontent.com/61985236/134768551-75408121-ca67-4966-9463-d759f69f5a25.png)

Hiểu sương sương nghĩa là nếu bạn có tài khoản thì có thể vào đường dẫn /devchat, đặc biệt hơn nếu có tài khoản admin thì có thể vào /admin. Hiện tại đương nhiên mình không thể vào trang admin được rồi, 
ngó devchat xem có gì

![image](https://user-images.githubusercontent.com/61985236/134768594-a9f1049f-714b-4279-84cf-5c907c803954.png)

Đoạn chat có đề cập đến NFKD => NFKD normalised exploit :)). Tiếp theo tạo một tài khoản và đăng nhập, thì có thêm trang profile cho phép thay đổi thông tin cá nhân

